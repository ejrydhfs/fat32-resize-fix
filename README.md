Gparted has a problem, which is not being able to resize fat32 partitions under 100 mb in size. this repo fixes that on its libparted dependency as follows: 

libparted can only reduce the cluster size at this point.  Start with the
cluster size of the existing file system and work downwards.  The existing
cluster size may be smaller than fat_min_cluster_size(), which is only a
preference when creating a file system: a FAT32 file system smaller than
~256 MiB must use 512 byte clusters to reach the mandatory 65525 clusters,
and mkfs.fat creates exactly that.  Therefore keep trying down to single
sector clusters, just like fat_calc_sizes() does when creating.

this has not been sumbitted yet to gparted because it needs further testing with larger fat partitions and also windows esp partitoinns

---

# Findings

## What was recompiled

**Yes, something was recompiled — GNU parted, not gparted.**

| Component | Action | Reason |
|---|---|---|
| GNU parted 3.6 → `libparted.so.2` + `libparted-fs-resize.so.0` | **rebuilt from source with the patch and installed system-wide** | contains the actual bug; GParted calls `ped_file_system_resize()` from here for all FAT grow/shrink |
| gparted 1.8.0 | **not recompiled** (source unchanged) | its source is correct: `src/fat16.cc` sets `fs.grow = FS::LIBPARTED; fs.shrink = FS::LIBPARTED` and `GParted_Core::resize_move_filesystem_using_libparted()` calls into libparted |
| dosfstools 4.2 | **untouched** | `mkfs.fat`/`fsck.fat` behave correctly (see below) |

GParted's FAT resize code path:

```
fat16.cc:  fs.grow/shrink = FS::LIBPARTED
  → GParted_Core::resize_plain()
  → GParted_Core::resize_move_filesystem_using_libparted()
  → ped_file_system_resize()            [libparted-fs-resize]
  → fat_resize() → fat_calc_resize_sizes()   ← the bug
```

## The bug (root cause)

`libparted/fs/r/fat/calc.c`, `fat_calc_resize_sizes()` only tried cluster sizes
from the current one down to `fat_min_cluster_size()` — which is **8 sectors
(4 KiB) for FAT32**. A FAT32 below ~256 MiB can only be spec-legal with 512 byte
clusters (65525 cluster minimum), so the loop body never ran and every resize
(grow *and* shrink) failed with:

```
GNU Parted cannot resize this partition to this size.  We're working on it!
```

dosfstools is not at fault: `mkfs.fat -F 32` on 100 MiB correctly creates 512 byte
clusters / 201 568 clusters, and `fsck.fat` validates it. Forcing 4 KiB clusters
would produce a 25 600 cluster "FAT32" that the Linux vfat driver misreads as
FAT16. Larger FAT32 volumes get 4 KiB clusters from `mkfs.fat`, which is exactly
why larger partitions could be resized before this fix.

The fix is in `parted-3.6-fat-resize-small-clusters.patch`: try cluster sizes from
the existing one down to single-sector clusters.

## Test results

### Test 1 — 100 MiB FAT32 (the original bug)

| Step | Result |
|---|---|
| Create 100 MiB FAT32 (`mkfs.fat -F 32`), 512 B clusters, 201 568 clusters | OK |
| Fill 100% with random data (`bigfile.bin` 103 198 720 B + 3 small files), `fsck.fat` clean | OK |
| **Before fix**: GParted grow 100 → 200 MiB | **FAILED** (`cannot resize this partition to this size`) |
| **After fix**: GParted grow 100 → 200 MiB (real UI) | **OK**, 403 217 clusters |
| Original 4 files md5-identical after resize | OK |
| Add 50 MB new data, remount, re-verify | OK (new file md5 stable, originals unchanged) |
| `fsck.fat` after all writes | clean |

### Test 2 — 1 GiB FAT32 (regression test, 4 KiB clusters)

Image: 8 GiB disk, MBR, 1 GiB FAT32 partition filled 100% with random data
(`bigfile.bin` 1 071 607 808 B + 3 small files; 261 627/261 627 clusters used),
md5 checksums recorded. All checks below compare against those checksums.

| Step | Result |
|---|---|
| Grow 1 → 2 GiB via GParted UI | **OK** — 523 260 clusters, `5 files, 261627` used (unchanged) |
| Originals md5-identical after grow | **YES** |
| Add 500 MB random data + 1 text file after grow | **OK** — 1 023 MiB free, writes accepted |
| New data survives remount (md5 stable) | **YES** |
| Originals still intact after the new writes | **YES** |
| Delete the new data | **OK**, only the 4 original files remain |
| Shrink 2 GiB → 1026 MiB (GParted's advertised *Minimum size*) | **FAILED** — see finding below |
| Data after the failed shrink | **intact** — `fsck.fat` clean, 5 files, 261 627 clusters, md5 identical; the failed operation left the file system untouched |
| Shrink 2 GiB → 1100 MiB (leaving headroom) | **OK** — 281 045 clusters, `fsck.fat` clean |
| Originals md5-identical after successful shrink | **YES** |

So the fix holds for larger partitions: growth works at both 512 B and 4 KiB
cluster sizes and is data-safe in both directions.

## Finding: shrinking to GParted's advertised minimum fails (separate, pre-existing bug)

When shrinking to exactly the minimum GParted offers (1026 MiB for 1 GiB of
data), the shrink aborts:

```
Shrink /dev/loop18p1 from 2.00 GiB to 1.00 GiB      ( ERROR )
  check file system ... fsck.fat -a -w -v            ( SUCCESS )
  shrink file system                                 ( ERROR )
    using libparted → libparted messages             ( ERROR )
      fat_table_alloc_cluster: no free clusters
```

(gdb backtrace: `fat_table_alloc_cluster` ← `write_fragments` (clstdup.c:330) ←
`fat_duplicate_clusters` ← `fat_resize` ← `ped_file_system_resize`)

Root cause, from the code and the debugger:

1. On resize, libparted shifts the cluster area by `start_move_delta` (510
   clusters here) so clusters stay aligned while the FAT size changes.
2. `fat_op_context_create_initial_fat()` marks **both** the in-place mapped used
   clusters (261 627) **and** the new file system clusters underlying the old
   metadata (510) as used → 262 137 of 262 138 clusters occupied, **zero free**.
3. Directory clusters are *always* relocated (`needs_duplicating()` returns true
   for `FAT_FLAG_DIRECTORY`), so at least one fresh cluster must be allocated →
   `fat_table_alloc_cluster: no free clusters`.

`fat_resize_constraint()` (the minimum size GParted displays) counts only
`used_clusters + total_dir_clusters` and does not reserve the shift overlap, so
GParted's "Minimum size" is not actually attainable.

This bug is **pre-existing upstream behaviour and independent of this repo's fix**:

* With the *stock, unpatched* `libparted-fs-resize.so.0.0.5` (extracted from
  Ubuntu's `libparted-fs-resize0t64_3.6-6`), the same shrink fails with the
  identical error.
* The patched code path is byte-equivalent for 4 KiB cluster file systems (the
  loop tries cluster size 8 first in both versions).

**Practical rule:** when shrinking FAT32, leave headroom of at least
(start_move_delta + relocated directory clusters) ≈ **5 MiB** above the used
data. With that headroom (1100 MiB target) the shrink completes and data
survives intact. A proper fix belongs upstream in `fat_resize_constraint()` /
the relocation planning.

The failed shrink is safe: it aborts during planning/relocation setup and leaves
the file system untouched (verified by fsck + md5 above).

## Evidence files

| File | What it shows |
|---|---|
| `parted-3.6-fat-resize-small-clusters.patch` | the fix |
| `gparted-after-resize-200MiB.png` | GParted after the successful 100 MiB → 200 MiB resize |
| `gparted-1g-grow-queued.png` | GParted queued "Grow /dev/loop18p1 from 1.00 GiB to 2.00 GiB" |
| `gparted-1g-shrink-error.png` | "An error occurred while applying the operations" for the minimum-size shrink |
| `gparted-shrink-failure-details.html` | GParted's saved operation details incl. the libparted error |
| `omp-session-export.txt` / `omp-session.jsonl` | exported omp session of this work (secrets redacted) |

## Rollback

```
sudo apt-get install --reinstall libparted2t64 libparted-fs-resize0t64
```

restores stock libraries.

## Still open

* Windows ESP partitions — not yet tested (see note above).
* Reporting the two libparted issues (cluster-size floor in
  `fat_calc_resize_sizes`, unattainable minimum in `fat_resize_constraint`)
  upstream to GNU parted.

Gparted has a problem, which is not being able to resize fat32 partitions under 100 mb in size. this repo fixes that on its libparted dependency as follows: 
libparted can only reduce the cluster size at this point.  Start with the
cluster size of the existing file system and work downwards.  The existing
cluster size may be smaller than fat_min_cluster_size(), which is only a
preference when creating a file system: a FAT32 file system smaller than
~256 MiB must use 512 byte clusters to reach the mandatory 65525 clusters,
and mkfs.fat creates exactly that.  Therefore keep trying down to single
sector clusters, just like fat_calc_sizes() does when creating.

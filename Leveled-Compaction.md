## Structure of the files
Files on disk are organized in multiple levels. We call them level-1, level-2, etc, or L1, L2, etc, for short. A special level-0 (or L0 fors short) contains files just flushed from in-memory write buffer (memtable). Each level (except level 0) is one data sorted run:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_structure.png)

Inside each level (except level 0), data is range partitioned into multiple SST files:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_files.png)

The level is a sorted run because keys in each SST file are sorted (See [Block-based Table Format](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format) as an example). To identify a position for a key, we first binary search the start/end key of all files to identify which file possibly contains the key, and then binary search inside the key to locate the exact position. In all, it is a full binary search across all the keys in the level.

All non-0 levels have target sizes. Compaction's goal will be to data size of those levels to be under the target. The size targets are usually exponentially increasing:
![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_targets.png)

## Compactions
Compaction triggers when number of L0 files reaches `level0_file_num_compaction_trigger`, files of L0 will be merged into L1. Normally we have to pick up all the L0 files because they usually are overlapping:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/pre_l0_compaction.png)

After the compaction, it may push the size of L1 to exceed its target:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/post_l0_compaction.png)

In this case, we will pick at least one file and merge it with the overlapping range of L2. The result files will be placed in L2:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/pre_l1_compaction.png)

If the results push the next level's size exceeds the target, we do the same as previously -- pick up a file and merge it into the next level:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/post_l1_compaction.png)

and then

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/pre_l2_compaction.png) 

and then

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/post_l2_compaction.png) 



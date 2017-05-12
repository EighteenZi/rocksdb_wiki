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

Multiple compactions can be executed in parallel if needed:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/multi_thread_compaction.png)

Maximum number of compactions allowed is controlled by `max_background_compactions`.

However, L0 to L1 compaction cannot be parallelized. In some cases, it may become a bottleneck that limit the total compaction speed. In this case, users can set `max_compactions` to more than 1. In this case, we'll try to partition the range and use multiple threads to execute it:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/subcompaction.png)

## Compaction Picking
When multiple levels trigger the compaction condition, RocksDB needs to pick which level to compact first. A score is generated for each level:

* For non-zero levels, the score is total size of the level divided by the target size. If there is already files picked that are being compacted into the next level, the size of those files are not included into the total size, because they will soon go away.

* for level-0, the score is the total number of files, divided by `level0_file_num_compaction_trigger`, or total size over `max_bytes_for_level_base`, which ever is larger. (if the file size is smaller than `level0_file_num_compaction_trigger`, compaction won't trigger from level 0, no matter how big the score is)

We compare the score of each level, and the level with highest score takes the priority to compact.

Which file(s) to compact from the level are explained in [[Choose Level Compaction Files]].

## Levels' Target Size
### `level_compaction_dynamic_level_bytes` is `false`
If `level_compaction_dynamic_level_bytes` is false, then level targets are determined as following: L1's target will be `max_bytes_for_level_base`. And then `Target_Size(Ln+1) = Target_Size(Ln) * max_bytes_for_level_multiplier * max_bytes_for_level_multiplier_additional[n]`. `max_bytes_for_level_multiplier_additional` is by default all 1.

For example, if `max_bytes_for_level_base = 123456`, `max_bytes_for_level_multiplier = 10` and `max_bytes_for_level_multiplier_additional` is not set, then size of L1, L2, L3 and L4 will be 16384, 163840, 1638400, and 16384000, respectively.  

### `level_compaction_dynamic_level_bytes` is `true`
Target size of the last level (`num_levels`-1) will always be actual size of the level. And then `Target_Size(Ln-1) = Target_Size(Ln) / max_bytes_for_level_multiplier`. We won't fill any level whose target will be lower than `max_bytes_for_level_base / max_bytes_for_level_multiplier `. These levels will be kept empty and all L0 compaction will skip those levels and directly go to the first level with valid target size.

For example, if `max_bytes_for_level_base` is 1GB, `num_levels=6` and the actual size of last level is 276GB, then the target size of L1-L6 will be 0, 0, 0.276GB, 2.76GB, 27.6GB and 276GB, respectively.

This is to guarantee a stable LSM-tree structure, which can't be guaranteed if `level_compaction_dynamic_level_bytes` is `false`. For example, in the previous example:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/dynamic_level.png)

We can guarantee 90% of data is stored in the last level, 9% data in the second last level. There will be multiple benefits to it. 
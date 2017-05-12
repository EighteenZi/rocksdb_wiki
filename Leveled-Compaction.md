# Structure of the files
Files on disk are organized in multiple levels. A special level-0 contains files just flushed from in-memory write buffer (memtable). Each level (except level 0) is one data sorted run:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_structure.png)

Inside each level (except level 0), data is range partitioned into multiple SST files:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_files.png)

The level is a sorted run because keys in each SST file are sorted (See [Block-based Table Format](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format) as an example). To identify a position for a key, we first binary search the start/end key of all files to identify which file possibly contains the key, and then binary search inside the key to locate the exact position. In all, it is a full binary search across all the keys in the level.

All non-0 levels have target sizes. Compaction's goal will be to data size of those levels to be under the target. The size targets are usually exponentially increasing:
![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_targets.png)


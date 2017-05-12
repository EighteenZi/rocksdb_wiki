# Structure of the files
Files on disk are organized in multiple levels. A special level-0 contains files just flushed from in-memory write buffer (memtable). Each level (except level 0) is one data sorted run:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_structure.png)

Inside each level (except level 0), data is range partitioned into multiple SST files:

![](https://github.com/facebook/rocksdb/blob/gh-pages-old/pictures/level_files.png)


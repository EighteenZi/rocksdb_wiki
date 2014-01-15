## Introduction
PlainTable (not yet committed to master branch)  is a format of SST files in RocksDB optimized for low query latency on pure-memory or really low-latency medias.
 
The reason for the high latency:
* A in-memory index is build to replace plain binary search by hash + binary search
* Bypassing block cache to void the costs of block copying and LRU cache maintaining.
* Avoid any memory copy when querying (by forcing mmap mode and not supporting compression and delta encoding)
 
Its limitation:
* File size needs to be smaller than 31 bits integer.
* Data compression is not supported
* Delta encoding is not supported
* Iterator.Prev() is not supported
* Non-prefix-based Seek() is not supported
* Table loading is slower for building indexes
* Only support mmap mode.
 
 
 
## Future Plan
* May consider to materialize the index to be a part of the SST file


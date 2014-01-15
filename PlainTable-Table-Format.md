### Introduction
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
 
 
 
### On-Disk Format
Here is the on disk format
 
    <beginning_of_file>
    [data row1]
    [data row1]
    [data row1]
    ...
    [data rowN]
    [Property Block]
    [Footer]                               (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
 
Format of property block and footer is the same as _BlockBasedTable format_(https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)
 
Two properties in property block are used to read data:
    data_size: the end of data part of the file
    fixed_ken_len: tells whether key is fixed length and what is the length if it is.
Each data row is encoded as:
    <beginning of a row>
    length of key: varint32 ^
    key bytes
    length of value: varint32
    value bytes
    <end of a row>
 
^ If keys are of fixed length, there will not be "length of key" field.
 
### Future Plan
 
* May consider to materialize the index to be a part of the SST file


### Introduction
PlainTable (not yet committed to master branch)  is one of RocksDB's SST file format optimized for low query latency on pure-memory or really low-latency media.
 
Advantages:
* An in-memory index is built to replace plain binary search with hash + binary search
* Bypassing block cache to avoid the overhead of block copy and LRU cache maintenance.
* Avoid any memory copy when querying (mmap)
 
Limitations:
* File size needs to be smaller than 31 bits integer. (TODO: we might as well lift this limit if index memory overhead is not a concern)
* Data compression is not supported
* Delta encoding is not supported
* Iterator.Prev() is not supported
* Non-prefix-based Seek() is not supported (TODO: sst is total ordered, normal seek is supported?)
* Table loading is slower for building indexes
* Only support mmap mode.
 
### File Format

    <beginning_of_file>
      [data row1]
      [data row1]
      [data row1]
      ...
      [data rowN]
      [Property Block]
      [Footer]                               (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
 
Format of property block and footer is the same as [BlockBasedTable format](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)
 
Two properties in property block are used to read data:

    data_size: the end of data part of the file

    fixed_ken_len: length of the key if all keys have the same length, 0 otherwise.

Each data row is encoded as:

    <beginning of a row>
      [length of key: varint32] ^
      key bytes
      length of value: varint32
      value bytes
    <end of a row>
 
^ If keys are of fixed length, there will not be "length of key" field.
 
### In-Memory Index Format [TODO: this section is a bit hard to read. Maybe restructure the content to make it cleaner. Also explain some of the decisions]

#### Basic Idea

In-memory Index is built to be as compact as possible. On top level, the index is a hash table with each bucket to be either offset in the file or a binary search index. The binary search is needed in two cases:

(1) Hash collisions: two or more prefixes are hashed to the same bucket

(2) Too many Keys for one prefix: need to speed-up the look-up inside one prefix.

#### Format

The index consists of two piece of memory: an array for hash buckets, and a buffer containing the binary searchable indexes (call it "binary search buffer" below, and "file" as the SST file). 

Key is hashed to buckets based on hash of its prefix (extracted using Options.prefix_extractor).

    +--------------+------------------------------------------------------+
    | Flag (1 bit) | Offset to binary search buffer or file (31 bits)     +
    +--------------+------------------------------------------------------+

If Flag = 0 and offset field equals to the offset of end of the data of the file, it means null - no data for this bucket; offset is smaller, it means there is only one prefix [TODO: what about prefix with less than 16 entries?] in the bucket, starting from that file offset. If Flag = 1, it means the offset is for binary search buffer. The format from that offset is shown below.

Starting from the offset of binary search buffer, a binary search index is encoded as following:

    <begin>
      number_of_records:  varint32
      record 1 file offset:  fixedint32
      record 2 file offset:  fixedint32
      ....
      record N file offset:  fixedint32
    <end>

where N = number_of_records-1. [TODO: N = number of records?] The offsets are in ascending order.

The reason for only storing 31-bit offset and use 1-bit to identify whether a binary search is needed is to make the index compact.

#### Index Look-up

To look up a key, first calculate prefix of the key using Options.prefix_extractor, and find the bucket for the prefix. If the bucket has no record on it (Flag=0 and offset is the offset of data end in file), the key is not found. Otherwise,

If Flag=0, it means there is only one prefix for the bucket and there are not many keys for the prefix, so the offset field points to the file offset of the prefix. We just need to do linear search from there.

If Flag=1, a binary search is needed for this bucket. The binary search indexes can be retrieved from the offset field. After the binary search, identify the index that is equal or larger than the look-up key. Say Mth. Both of (M-1)th (if exists) and Mth can contain the prefix of the look-up key. Determine which one we should start the linear search with, by comparing their prefixes. [TODO: rephrase the above]

#### Building the Index

When Building indexes, scan the file. For each key, calculate its prefix, remember (hash value of the prefix, offset) information for the (16n+1)th row of each prefix (n=0,1,2...), starting from the first one. 16 the maximum number of rows need to check in the linear search following the binary search. Based on the number of prefixes, determine an optimal bucket size. Allocate exact buckets and binary search buffer needed and fill in the indexes according to the bucket size.

#### Bloom Filter
A bloom filter on prefixes can be configured for queries. User can config how many bits are allocated for every prefix. When doing the query (Seek() or Get()), bloom filter is checked and filter out non-existing prefixes before looking up the indexes.

### Future Plan
 
* May consider to materialize the index to be a part of the SST file
* May build extra more sparse indexes to enable general iterator seeking.

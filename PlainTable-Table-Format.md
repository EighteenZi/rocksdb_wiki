### Introduction
PlainTable (not yet committed to master branch)  is a format of SST files in RocksDB optimized for low query latency on pure-memory or really low-latency media.
 
Advantage:
* An in-memory index is built to replace plain binary search with hash + binary search
* Bypassing block cache to avoid the overhead of block copy and LRU cache maintenance.
* Avoid any memory copy when querying (mmap)
 
Limitation:
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

#### Basic Idea [TODO: maybe explain the key idea as hash index + binary search index if prefix collides or there are simply too many entries within a single prefix] 

In-memory Index is built to be as compact as possible. On top level, the index is a hash table with each bucket to be an array list [TODO: array list sounds very confusing. How about just 'a binary search index']. The hash table is implemented in such a way: [TODO: maybe remove the following (1)(2)(3), as they will be discussed more clearly later]

(1) Pointers are implemented as file offset or buffer offset to use only 31 bits

(2) In hash table, the key is stored as a file offset to the row so that the key can be read from there

(3) Because of (1) and (2), for each hash table entry, it is only an offset to the file.

#### Format

The index consists of two piece of memory: an array for hash buckets, and a buffer containing the binary searchable indexes (call it "buffer" below, and "file" as the SST file). 

Key is hashed to buckets based on hash of its prefix (extracted using Options.prefix_extractor).

    +--------------+----------------------------------------+
    | Flag (1 bit) | Offset to buffer or file (31 bits)     +
    +--------------+----------------------------------------+

If Flag = 0 and offset field equals to the offset of end of the data of the file, it means null - no data for this bucket; offset is smaller, it means there is only one record [TODO: what about prefix with less than 16 entries?] of the bucket, which is for row(s) starting from that position. If Flag = 1, it means the offset is for buffer. The format from that offset is shown below.

Starting from the offset of buffer, a binary search index is encoded as following:

    <begin>
      number_of_records:  varint32
      record 1 file offset:  fixedint32
      record 2 file offset:  fixedint32
      ....
      record N file offset:  fixedint32
    <end>

where N = number_of_records-1. [TODO: N = number of records?] The offsets are in ascending order.

#### Index Look-up

Based on the index format, look-up procedure is straight-forward. To look up a key, first calculate prefix of the key using Options.prefix_extractor. Find the bucket for it. If the bucket has no record on it (Flag=0 and offset is the offset of data end in file), it can declared not found. Otherwise,

If Flag=0, go and check the key pointed by the offset of the bucket. Keep comparing the next key of rows starting from this point. If a key matches the prefix of the look-up key and is equal or larger, return it. Otherwise, keep searching the next key as long as the prefix matches.

If Flag=1, go to the area it points to in the buffer, doing a binary search. If an exact key match is identified, return it. Otherwise, identify Mth record where key is between keys pointed by Mth record and (M+1)th record. Both of Mth and (M+1)th can contain the prefix of the look-up key. Identify which one (if both matches, us M) and start linear search in file from there like Flag=0 case. [TODO: rephrase the above]

#### Building the Index

When Building indexes, scan the file. For each key, calculate prefix, record (hash value of the prefix, offset) for the (16n+1)th [TODO: explain the trade off when picking 16, which is the maximum number of linear scans]  row of each prefix (n=0,1,2...). Also count number of prefixes. Based on the number of prefixes, determine an optimal bucket size. Allocate exact buckets and buffer needed and fill in the indexes according to the bucket size.

[TODO: also add a brief section about prefix bloom filter]
### Future Plan
 
* May consider to materialize the index to be a part of the SST file


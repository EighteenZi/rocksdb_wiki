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
 
Format of property block and footer is the same as [BlockBasedTable format](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)
 
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
 
### In-Memory Index Format

#### Basic Idea

In-memory is built to be as compact as possible. On top level, the index is a hash table with each bucket to be an array list. The hash table is implemented in such a way:

(1) Pointers are implemented as file offset or buffer offset to use only 31 bits

(2) In hash table, the key is stored as a file offset to the row so that the key can be read from there

(3) Because of (1) and (2), for each hash table entry, it is only an offset to the file.

#### Format

The index consists of two piece of memory: an array for buckets, and a buffer containing the array lists (call it "buffer" below, and "file" as the SST file). 

Key is hashed to buckets based on hash of its prefix (extracted using Options.prefix_extractor).

  +--------------+----------------------------------------+
  | Flag (1 bit) | Offset to buffer or file (31 bits)     +
  +--------------+----------------------------------------+

If Flag = 0 and offset field equals to the offset of end of the data of the file, it means null - no data for this bucket; offset is smaller, it means there is only one record of the bucket, which is for row(s) starting from that position. If Flag = 1, it means the offset is for buffer. The format from that offset is shown below.

Starting from the offset of buffer, array list of entries of the buckets is encoded as following:

    <begin>
      number_of_records:  varint32
      record 1 file offset:  fixedint32
      record 2 file offset:  fixedint32
      ....
      record N file offset:  fixedint32
    <end>

where N = number_of_records-1. The offsets are in ascending order.

#### Index Lookup

Based on the index format, lookup procedure is straight-forward. To look up a key, first calculate prefix of the key using Options.prefix_extractor. Find the bucket for it. If the bucket has no record on it (Flag=0 and offset is the offset of data end in file), it can declared not found. Otherwise,

If Flag=0, go and check the key pointed by the offset of the bucket. Keep compare keys of rows starting from this point. If a key matches the prefix of the lookup key and is equal or larger, return it. Otherwise, keep searching the next key as long as the prefix matches.

If Flag=1, go to the area it points to in the buffer, doing a binary search. If an exact key match is identified, return it. Otherwise, identify Mth record where key is between keys pointed by Mth record and (M+1)th record. Both of Mth and (M+1)th offset can contain the prefix of the look-up key. Identify the one matching the prefix (if both matches, us M) and start linear search in file from there.

#### Building the Index

When Building indexes, scan the file, record hash value and offset for the first, and (16n+1)th row of each prefix. Also count number of prefixes. Based on the number of prefixes, determine an optimal bucket size. Fill in the indexes according to the bucket size.


### Future Plan
 
* May consider to materialize the index to be a part of the SST file


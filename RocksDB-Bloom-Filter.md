### What is Bloom Filter
A bloom filter is a bit array that could tell whether an arbitrary key may exist in a set of keys.

Say that there is a set of keys, an algorithm could be applied to create a bit array called bloom filter. Then, given an arbitrary key, this bit array could tell whether a key “may exist” or “definitely not exist” in the original key sets.

[Here](http://en.wikipedia.org/wiki/Bloom_filter) is a detailed explanation for how bloom filter works.

In RocksDB, every SST file contains a bloom filter. It is used to decide if we need to go into the file to find the exact key.

#### Life Cycle
In RocksDB, each SST file has a bloom filter, which is created as soon as the file is written. This bloom filter is stored as a part of SST file. There is no difference when building bloom filter for files in different levels.

There is no operation to combine bloom filters. Bloom filters can only be created from a set of keys. When we combine two SST files, new bloom filter is created from keys of new file.

When we open a SST file, the bloom filter is opened and loaded in memory. If 

    options:: allow_mmap_reads=true

The bloom filter is memory mapped.

When the SST file is closed, the bloom filter is removed from memory. But if

    BlockBasedTableOptions::cache_index_and_filter_blocks=true,  

the bloom filter may be cached in block cache.

#### Memory Usage
The bloom filter needs all keys to fit in memory to build it. It seems at first sight that building bloom filter for large SST file is impossible. But in block based table, there is no need to worry because bloom filter is created for each data block that just contains a small range of keys.
 
Details for format of block based table’s filter could be found [here](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format#filter-meta-block).

The block size is defined in 

    Options::block_size

The default value is 4K. When building a SST file, key-value pairs are added in a sequence, when there is enough k-v pairs ( < 4K), RocksDB would create bloom filter for them and write it to file. So only a very small fraction of keys is stored in memory to build the bloom filter.

We are also working on a new bloom filter for block based table that contains a filter for all keys in SST file. It could improve RocksDB read performance but requires storing hashes of all the keys when building it. Details could be found in next section.

#### New bloom filter format
[Original](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format#filter-meta-block) bloom filter creates filter for each "data block", thus requires little memory when building it. 

We are working on a new bloom filter(called full filter) that contains a filter for all keys in the SST file. This could improve reading performance because it avoids traveling in complicated SST format. But it takes more memory when building because all keys in SST file is required. We have optimization to store hashes of keys in memory. But users still needs to think twice when using it for large SST files.

Users could specify which kind of bloom filter to use in

    tableOptions::FilterBlockType

It use the original bloom filter by default.

The full filter block is formatted as follows:

    [filter]
    [num probes]        : 1 byte
    [num blocks]        : 4 bytes

The filter is a big bits array that could be used to check for all keys in SST file. 
num probes is the number of hash functions used to create bloom filter. In original bloom filter format, it is attached at the end of each filter. (Actually it is the same with full filter)
num blocks is a parameter used inside of filter's algorithm. It has no relation to blocks in SST file.





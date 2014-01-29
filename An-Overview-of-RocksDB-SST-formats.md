## Choosing a table format

Rocksdb supports different types of tables (also known as `sst`, sorted static table), but how to choose the table format that fits your need best?

Right now we have two types of tables: "plain table" and "block based table".

### Block-based table ###

this is the default table type that we inherited from [LevelDB](http://leveldb.googlecode.com/svn/trunk/doc/index.html), which was designed for storing data in hard disk or flash device.

In block-based table, data is chucked into (almost) fix-sized blocks (default block size is 4k). Each block, in turn, keeps a bunch of entries.

When storing data, we can compress and/or encode data efficiently within a block, which often resulted in a much smaller data size compared with the raw data size.

As for the record retrieval, we'll first locate the block where target record may reside, then read the block to memory, and finally search that record within the block. Of course, to avoid frequent reads of the same block, we introduced the `block cache` to keep the loaded blocks in the memory.

For more information about block-based table, please read this wiki: [Rocksdb BlockBasedTable Format](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)

### Plain table ###

## Example
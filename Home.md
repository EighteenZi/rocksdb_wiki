# Welcome to RocksDB
RocksDB is a C++ library providing an embedded key-value store, where keys and values are arbitrary byte streams. It was developed at Facebook based on LevelDB and provides backwards-compatible support for LevelDB APIs.

RocksDB is optimized for Flash with extremely low latencies. RocksDB uses a Log Structured Database Engine for storage, written entirely in C++. A Java version called RocksJava is currently in development.

RocksDB features highly flexible configuration settings that may be tuned to run on a variety of production environments, including pure memory, Flash, hard disks or HDFS. It supports various compression algorithms and good tools for production support and debugging.

## Features
* Designed for application servers wanting to store up to a few terabytes of data on locally attached Flash drives or in RAM
* Optimized for storing small to medium size key-values on fast storage -- flash devices or in-memory
* Scales linearly with number of CPUs so that it works well on ARM processors


## Getting Started
For a complete Table of Contents, see the sidebar to the left. Most readers will want to start with the [Overview](https://github.com/barnaby0101/sandbox/wiki/Overview) and the [Basic Operations](https://github.com/barnaby0101/sandbox/wiki/RocksDB-Introduction) section of the Developer's Guide. 


## News 
* [RocksDB v3.3 released](https://github.com/facebook/rocksdb/releases/tag/rocksdb-3.3) (July, 2014)
* [RocksDB v3.2 Released](http://rocksdb.org/blog/647/rocksdb-3-2-release/) (June, 2014)
* [RocksDB v3.1 Released](http://rocksdb.org/blog/575/rocksdb-3-1-release/) (May, 2014)
* [RocksJava in development](https://github.com/facebook/rocksdb/wiki/RocksJava-Basics) (May, 2014)
* [RocksDB MeetUp](http://rocksdb.org/blog/323/the-1st-rocksdb-local-meetup-held-on-march-27-2014/) (March, 2014)



## Project History
* [The History of RocksDB](http://rocksdb.blogspot.com/2013/11/the-history-of-rocksdb.html)
* [Under the Hood: Building and open-sourcing RocksDB](https://www.facebook.com/notes/facebook-engineering/under-the-hood-building-and-open-sourcing-rocksdb/10151822347683920).

## Links 
* [Sample Applications](https://github.com/facebook/rocksdb/tree/master/examples)
* [Official Blog](http://rocksdb.org/blog/)
* [Stack Overflow: RocksDB](https://stackoverflow.com/questions/tagged/rocksdb)
* [Talks](https://github.com/facebook/rocksdb/wiki/Talks)

## Contact 
* [Public Developer's Discussion Group](https://www.facebook.com/groups/rocksdb.dev/)
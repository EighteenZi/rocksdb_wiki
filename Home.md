# Welcome to RocksDB
RocksDB is a C++ library providing an embedded key-value store, where keys and values are arbitrary byte streams. It was developed at Facebook based on LevelDB and provides backwards-compatible support for LevelDB APIs.

RocksDB is optimized for Flash with extremely low latencies. RocksDB uses a Log Structured Database Engine for storage, written entirely in C++. A Java version called RocksJava is currently in development.

RocksDB features highly flexible configuration settings that may be tuned to run on a variety of production environments, including pure memory, Flash, hard disks or HDFS. It supports various compression algorithms and good tools for production support and debugging.

## Features
* Designed for application servers wanting to store up to a few terabytes of data on locally attached Flash drives or in RAM
* Optimized for storing small to medium size key-values on fast storage -- flash devices or in-memory
* Scales linearly with number of CPUs so that it works well on processors with many cores

## Features Not in LevelDB
RocksDB introduces dozens of new major features. See [the list of features not in LevelDB](https://github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB).


## Getting Started
For a complete Table of Contents, see the sidebar to the left. Most readers will want to start with the [Overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics) and the [Basic Operations](https://github.com/facebook/rocksdb/wiki/Basic-Operations) section of the Developer's Guide. Also check [[RocksDB FAQ]] and [[RocksDB Tuning Guide]].

## Reporting bugs and asking for help
If you will run into any issues then please use [these guidelines] (https://github.com/facebook/mysql-5.6/wiki/Reporting-bugs-and-asking-for-help) to report bugs and ask for help.

## Blog 
* Check out our blog at [rocksdb.org/blog](http://rocksdb.org/blog)

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
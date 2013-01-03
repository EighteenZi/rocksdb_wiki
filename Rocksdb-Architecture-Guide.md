**# Introduction**
  
The rocksdb project started at [Facebook](https://www.facebook.com/Engineering) as an experiment to  develop an efficient database software that can realize the full potential of storing data on flash drives. It is a C++ library and can be used to store keys-and-values where keys and values are arbitrary size byte streams. It has support for atomic reads and atomic writes but not general purpose transactions. It has highly flexible configurable settings that can be tuned to run on a variety of production environments: it can be configured to run on data on pure memory, flash, hard disks or on HDFS. It can be setup to support various compression algorithms and good tools for production support and debugging. It also has Java apis that can be used by applications written in Java.
  
Some portions of the code has been inherited from the open source [leveldb](https://code.google.com/p/leveldb/) project. If you have an application that uses leveldb, you should be able to use rocksdb without changing a single line of code in your application.
 

**# Assumptions and Goals**

**## Performance:** The primary design point for rocksdb is that it should be performant for fast storage. It should be able to exploit the full potential of high read/write rates offered by flash or RAM-memory subsystems.  It should support efficient point lookups as well as range scans. It should be configurable to support high random-read workloads, high update workloads or a combination of both.

**## Production support:** Rocksdb should be designed in such a way that it has built-in support for tools and utilities that help deployment and debugging in production environments. Most major parameters should be fully tunable so that it can be used by different applications on different hardware.

**## Backward Compatibility:** Newer versions of this software should be backward compatible, so that existing applications do not need to change when upgrading to newer releases of rocksdb. 




**# Read write apis**
  * Puts
  -- batch puts
  -- disable wal
  -- async puts, fsync and fdatasync
  -- Iterators and snapshots
  -- batch commit to transaction log and manifest updates
  -- binary search for overlapping files for every level
  checksums for reads (default false)
  hardware assists
  bloom filters
  shared block cache
  ReadOnly mode

**# Disk Format**
  .sst files for data
  .log files for trasactions
  manifest_file for database versions
  LOG* for server information logs

**# Compactions**
  -- multi-threaded
  -- thread pool per environment
  -- priority queues for merge sort
  -- user defined hook for implementing ttl, sanity checks, etc
  -- avoid compression for two levels, snappy, bzip, zlib

**# Incremental Backups**
  GetLiveFiles
  GetUpdatesSince
    -- wals are archived

**# Environments**
  posix (production)
  hdfs environment (prototype)

**# Tools and Tests**
  sst_dump
  manifest_dump
  compact database, change number of levels
  stress test

**# java api**

### Author: Dhruba Borthakur
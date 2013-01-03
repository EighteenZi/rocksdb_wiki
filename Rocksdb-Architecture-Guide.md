**# Introduction**
  The rocksdb project started at [Facebook](https://www.facebook.com/Engineering) as an experiment to  develop an efficient database software that can realize the full potential of storing data on flash drives. It is a C++ library and can be used to store keys-and-values where keys and values are arbitrary size byte streams. It has support for atomic reads and atomic writes but not general purpose transactions. It has highly flexible configurable settings that can be tuned to run on a variety of production environments: it can be configured to run on data on pure memory, flash, hard disks or on HDFS. It can be setup to support various compression algorithms and good tools for production support and debugging. It also has Java apis that can be used by applications written in Java.
  Some portions of the code has been inherited from the open source [leveldb](https://code.google.com/p/leveldb/) project. If you have an application that uses leveldb, you should be able to use rocksdb without changing a single line of code in your application.
 

**# Assumptions and Goals**
  -- good for fast storage (flash)
  -- good for random reads
  -- tradeoff between read ampl, write ampl
  -- range scans and point lookups
  -- production settting needs configurable param
     num levels, size of files, size of dats in a level, etc

**# Disk Format**
  .sst files for data
  .log files for trasactions
  manifest_file for database versions
  LOG* for server information logs

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

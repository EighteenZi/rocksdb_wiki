**1. Introduction**
  
The rocksdb project started at [Facebook](https://www.facebook.com/Engineering) as an experiment to  develop an efficient database software that can realize the full potential of storing data on flash drives. It is a C++ library and can be used to store keys-and-values where keys and values are arbitrary size byte streams. It has support for atomic reads and atomic writes but not general purpose transactions. It has highly flexible configurable settings that can be tuned to run on a variety of production environments: it can be configured to run on data on pure memory, flash, hard disks or on HDFS. It has support for various compression algorithms and good tools for production support and debugging. 
  
Some portions of the code has been inherited from the open source [leveldb](https://code.google.com/p/leveldb/) project. If you have an application that uses leveldb, you should be able to use rocksdb by changing a few line of code in your application.
 
**2. Assumptions and Goals**

**## Performance:** The primary design point for rocksdb is that it should be performant for fast storage. It should be able to exploit the full potential of high read/write rates offered by flash or RAM-memory subsystems.  It should support efficient point lookups as well as range scans. It should be configurable to support high random-read workloads, high update workloads or a combination of both.

**## Production support:** Rocksdb should be designed in such a way that it has built-in support for tools and utilities that help deployment and debugging in production environments. Most major parameters should be fully tunable so that it can be used by different applications on different hardware.

**## Backward Compatibility:** Newer versions of this software should be backward compatible, so that existing applications do not need to change when upgrading to newer releases of rocksdb. 

## **3. Read write apis** 

Gets, Puts and Write-Batch
Rocksdb is a data store that stores arbitrary keys and values. The keys and values are treated as pure byte streams. There is no limit to the size of a key or a value. There is a Get api that allows an appliaction to fetch a single key-value from the database. A MultiGet api allows an application to retrieve  a bunch of keys from the database, all the keys-values returned va a MultiGet call are consistent with one-another 

Iterators and Snapshots
All data in the database is logically arranged in sorted order. An application can specify a Key-Comparison method that specifies a total ordering of keys. An Iterator api allows an application to do a RangeScan on the database. The Iterator can seek to a specified key and then the application can start scanning one key at a time from that point. The Iterator api can also be used to do a reverse iteration of the keys in the database. A consistent-point-in-time view of the database is created when the Iterator is created. Thus, all keys returned via the Iterator are from a consistent view of the database.

A Snapshot api allows an application to create a point-in-time view of a database. The Get and Iterator apis can be used to read data from a specified snapshot. In some sense, a Snapshot and an Iterator both provide a point-in-time view of the database. But their implementations are different. Short lived scans are best done via an Iterator while long-running scans are better used via a Snapshot. A Iterator keeps a reference count on all underlying files that correspond to that point-in-time-view of the database, these files will not get deleted till the Iterator is released. On the other hand, a Snapshot does not prevent file deletions; instead the compaction process understands the existence of snapshots and promises to never delete a key that is visible in any existing snapshot.

Snapshots are not persisted: a reload of the rocksdb library (via a server restart) would release all pre-existing snapshots.

Prefix Iterators
Most LSM engines cannot support an efficient RangeScan api because it needs to look into every data file. But most applications do not do random scans of key ranges in the database; instead applications typically scan within a key-prefix. Rocksdb uses this to its advantage. There is a PrefixTransform api that allows an application to specify a key-prefix. Rocksdb uses this to store blooms for this prefix and can avoid looking into data files when a RangeScan is performed within that prefix. 

Updates
There is a Put api that can insert a single key-value to the database. A WriteBatch api allows multiple keys-values to be inserted atomically into the database. The database guarantees that all the keys-values in a single WriteBatch call will either be inserted into the database or none of them will be inserted into the database. 

Persistency
Rocksdb has a transaction log. All Puts are stored in a in-memory buffer called the memtable as well as optionally inserted into a transaction log. Each Put has a bunch of flags specified via WriteOptions. The WriteOptions can specify whether the Put should be inserted into the transaction log. Also, the WriteOptions can specify whether a sync call is issued to the transaction log before a Put is declared to be committed. 

Iternally, rocksdb uses a batch-commit mechanism to batch transactions into the transaction log so that a sync can potentially commit more than a single transaction.

Fault Tolerance
Rocksdb uses a checksum to detect corruptions in the storage. These checksums are for each block; a block is typically anywhere between 4K to 128K in size. A block, once written to storage, is never modified. Rocksdb will dynamically detect hardware support for checksum computations and avail that support wherever available. 

Multithreaded Compactions
Compactions are needed to remove multiple copies of the same key that can occur if an application overwrtes an existing key. Compactions also processes deletions of keys. Compactions can occur in multiple threads if configured appropriately. The overall throughput of a LSM database is directly related to the speed at which compaction can occur, especially when the data is stored in fast storage like SSD or RAM; otherwise it is an unstable system. For this reason, rocksdb can be configured to issue compaction requests from multiple threads concurrently. It is observed that sustained write rates increase by a factor of 10 with multi-threaded compaction when the database is on SSDs compared to single-threaded compactions.

The 
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
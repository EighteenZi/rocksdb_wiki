# 1. Introduction
  
The rocksdb project started at [Facebook](https://www.facebook.com/Engineering) as an experiment to  develop an efficient database software that can realize the full potential of storing data on flash drives. It is a C++ library and can be used to store keys-and-values where keys and values are arbitrary size byte streams. It has support for atomic reads and atomic writes but not general purpose transactions. It has highly flexible configurable settings that can be tuned to run on a variety of production environments: it can be configured to run on data on pure memory, flash, hard disks or on HDFS. It has support for various compression algorithms and good tools for production support and debugging. 
  
Some portions of the code has been inherited from the open source [leveldb](https://code.google.com/p/leveldb/) project. If you have an application that uses leveldb, you should be able to use rocksdb by changing a few line of code in your application.
 
# 2. Assumptions and Goals

## Performance:
The primary design point for rocksdb is that it should be performant for fast storage. It should be able to exploit the full potential of high read/write rates offered by flash or RAM-memory subsystems.  It should support efficient point lookups as well as range scans. It should be configurable to support high random-read workloads, high update workloads or a combination of both.

## Production support:
Rocksdb should be designed in such a way that it has built-in support for tools and utilities that help deployment and debugging in production environments. Most major parameters should be fully tunable so that it can be used by different applications on different hardware.

## Backward Compatibility:
Newer versions of this software should be backward compatible, so that existing applications do not need to change when upgrading to newer releases of rocksdb. 

# 3. Read write apis

## Gets, Puts and Write-Batch
Rocksdb is a data store that stores arbitrary keys and values. The keys and values are treated as pure byte streams. There is no limit to the size of a key or a value. There is a Get api that allows an appliaction to fetch a single key-value from the database. A MultiGet api allows an application to retrieve  a bunch of keys from the database, all the keys-values returned va a MultiGet call are consistent with one-another 

Iterators and Snapshots
All data in the database is logically arranged in sorted order. An application can specify a Key-Comparison method that specifies a total ordering of keys. An Iterator api allows an application to do a RangeScan on the database. The Iterator can seek to a specified key and then the application can start scanning one key at a time from that point. The Iterator api can also be used to do a reverse iteration of the keys in the database. A consistent-point-in-time view of the database is created when the Iterator is created. Thus, all keys returned via the Iterator are from a consistent view of the database.

A Snapshot api allows an application to create a point-in-time view of a database. The Get and Iterator apis can be used to read data from a specified snapshot. In some sense, a Snapshot and an Iterator both provide a point-in-time view of the database. But their implementations are different. Short lived scans are best done via an Iterator while long-running scans are better used via a Snapshot. A Iterator keeps a reference count on all underlying files that correspond to that point-in-time-view of the database, these files will not get deleted till the Iterator is released. On the other hand, a Snapshot does not prevent file deletions; instead the compaction process understands the existence of snapshots and promises to never delete a key that is visible in any existing snapshot.

Snapshots are not persisted: a reload of the rocksdb library (via a server restart) would release all pre-existing snapshots.

## Prefix Iterators
Most LSM engines cannot support an efficient RangeScan api because it needs to look into every data file. But most applications do not do random scans of key ranges in the database; instead applications typically scan within a key-prefix. Rocksdb uses this to its advantage. There is a PrefixTransform api that allows an application to specify a key-prefix. Rocksdb uses this to store blooms for this prefix and can avoid looking into data files when a RangeScan is performed within that prefix. 

## Updates
There is a Put api that can insert a single key-value to the database. A WriteBatch api allows multiple keys-values to be inserted atomically into the database. The database guarantees that all the keys-values in a single WriteBatch call will either be inserted into the database or none of them will be inserted into the database. 

## Persistency
Rocksdb has a transaction log. All Puts are stored in a in-memory buffer called the memtable as well as optionally inserted into a transaction log. Each Put has a bunch of flags specified via WriteOptions. The WriteOptions can specify whether the Put should be inserted into the transaction log. Also, the WriteOptions can specify whether a sync call is issued to the transaction log before a Put is declared to be committed. 

Iternally, rocksdb uses a batch-commit mechanism to batch transactions into the transaction log so that a sync can potentially commit more than a single transaction.

## Fault Tolerance
Rocksdb uses a checksum to detect corruptions in the storage. These checksums are for each block; a block is typically anywhere between 4K to 128K in size. A block, once written to storage, is never modified. Rocksdb will dynamically detect hardware support for checksum computations and avail that support wherever available. 

## Multithreaded Compactions
Compactions are needed to remove multiple copies of the same key that can occur if an application overwrites an existing key. Compactions also processes deletions of keys. Compactions can occur in multiple threads if configured appropriately. The overall throughput of a LSM database is directly related to the speed at which compaction can occur, especially when the data is stored in fast storage like SSD or RAM; otherwise it is an unstable system. Rocksdb can be configured to issue compaction requests from multiple threads concurrently. It is observed that sustained write rates can increase upto a factor of 10 with multi-threaded compaction when the database is on SSDs compared to single-threaded compactions.

Level Style Compaction and Universal Style Compaction
The entire database is stored in a bunch of files. When a memtable is full, its contents is written out to a file in Level-0. Rocksdb removes duplicates and overwritten keys in the memtable when it is flushed to a file in L0. Periodically, some files are read in and merged to form larger files. This is called Compaction. 

In the Universal Style Compaction, a few files in L0 and merged and is written back into L0. In the Level Style Compactions, a few files in L0 are merged with the corresponding files in L1 and the output is written to new files in L1.

There is a MANIFEST file in the database that records all the files that make up the database. The Compaction process adds new files and deletes existing files from the database, and it makes these operations persistent by recording them in the MANIFEST file. Transactions to be recorded in the MANIFEST file uses a batch-commit algorithm to amortize the cost of repeated syncs to the MANIFEST file.

## Compaction Filter
There are times when an application would like to process keys at compaction time. For example, a database that has inherent support for Time-To-Live (ttl) can optionally remove keys that are expired. This can be done via a application defined Compaction Filter. The rocksdb Compaction Filter gives control to the application to modify the value of a key or to entirely drop a key as part of the compaction process.

## ReadOnly Mode
There are times when an application wants to open a database for reading only. It can open the database in ReadOnly mode, the database guarantees that the application won't be able to modify anything in the database.  

## Database Debug Logs
The rocksdb database software writes detailed logs to a file named LOG*. These are mostly used for debugging and analyzing a running system. There is a configuration to allow rolling this LOG at a specified periodicity.

## Data Compression
Rocksdb supports snappy, zlib and bzip2 compression. rocksdb can be configured to support different compression algorithms at data in different levels. Typically, 90% of data in in the L-max level. A typical installation might configure no-compression for levels L0-L2, snappy compression for the mid levels and zlip compression for Lmax.

## Full Backups, Incremental Backups and Replication
Rocksdb has support for full backups and incremental backups. Rocksdb is an LSM database engine, so datafiles once created are never overwritten and this makes it easy to extract a list of file-names that correspond to a point-in-time snapshot of the database contents. The api DisableFileDeletions instructs rocksdb to not delete data files. Compactions will continue to occur, but files that are not needed by the database will not be deleted. An backup application can then invoke the api GetLiveFiles to retrieve the list of files and copy them to a backup location. Once the backup is complete, the application should invoke EnableFileDeletions; the database is now free to reclaim all the files that are not needed any more. 

Incremental Backups and Replication needs to be able to find and tail all the recent changes to the database. The api GetUpdatesSince allows an application to tail the rocksdb transaction log. It can continuously fetch transactions from the rocksdb transaction log and apply them to a remote replica or a remote backup. 

A replication system typically wants to annotate each Put with some arbitrary metadata. This metadata maybe used to detect loops in the replication pipeline. It can also be used to timestamp and sequence transactions. For this purpose, rocksdb supports a api called PutLogData that an application can use to annotate each Put with metadata. This metadata is stored only in the transaction log and is not stored in the data (sst) files. The metadata inserted via PutLogData can be retrieved via the GetUpdatesSince api.

rocksdb transaction logs are created in the database directory. When a log file is no longer needed, it is moved to the archive directory. The reason for the existence of the archive directory is because a replication stream that is falling behind might need to retrieve transactions from a log file that is way in the past. The api GetSortedWalFiles returns a list of all transaction log files.

## Support for multiple embedded databases in the same process
A common use-case for applications that use rocksdb is that they inherently partition their data set into multiple logical partitions or shards. This technique is helpful for application load balancing and fast recovery from faults. This means that a single server process should be able to operate multiple rocksdb databases simultaneously. This is done via an environment object named Env. Among other things, a thread pool is associated with an Env. If applications want to share a common thread pool (for background compactions) among multiple database instances, then it should use the same Env object for opening those databases.

Similarly, there is support for multiple database instances to share the same block cache.

## Block Cache -- Compressed and Uncompressed Data
Rocksdb uses a LRU cache for blocks to serve reads. The block cache is partitions into two individual caches: the first one caches uncompressed blocks and the second one caches compressed blocks in RAM. If a compressed block cache is configured, then the database intelligently avoids caching data in the OS buffers.

## Tools
There are a number of interesting tools that are used to support a database in production. The sst_dump utility dumps all the keys-values in a sst file.  The ldb tool can put, get, scan the contents of a database. ldb can also dump contents of the MANIFEST, it can also be used to change the number of configured levels of the database. It can be used to manually compact a database.

## Tests
There are a bunch of unit tests that test specific features of the database. The db_stress test is used to validate data correctness at scale.

## Support for External Compaction Algorithms
The performance of an LSM database has a significant dependency on the compaction algorithm and implementation. rocksdb has two supported compaction algorithms: LevelStyle and UniversalStyle. But we would like to enable the large community of developers to develop and experiment with other compaction policies. For this reason, rocksdb has appropriate hooks to switch off the inbuilt compaction algorithm and has other apis to allow an application to operate their own compaction algorithms. Options.disable_auto_compaction, if set, disables the inbuilt compaction algorithm. The GetLiveFilesMetaData api allows an external component to look at every data file in the database, decide on which data files to merge and compact, and the DeleteFile api allows it to delete data files that are deemed obsolete.

## Non-blocking database access
There are certain applications that are architected in such a way that they would like to retrieve data from the database only if that data retrieval call is non-blocking, i.e. the data retrieval call does not have to read in data from storage. Rocksdb caches a portion of the database in the block cache and these applications would like to retrieve the data only if it is found in this block cache. If this call does not find the data in the block cache then rocksdb returns an appropriate error code to the application. The application can then schedule a normal Get/Next operation understanding that fact that this data retrieval call could potentially block for IO from the storage (maybe in a different thread context).

## Stackable DB
Rocksdb come with a built-in wrapper-mechanism to layer additional functionality. This functionality is encapsulated by an api called StackableDB. For example, the time-to-live functionality is implemented by a StackableDB and is not part of the core rocksdb api. This approach keep the code modularized and clean.

## Memtables:
### Pluggable Memtables:
The default implementation of the memtable for rocksdb is a skiplist. The skiplist is an sorted set. The sorted set is a necessary construct when the workload interleaves writes with range-scans. But there are some applications that do not interleave their writes and scans and there are other applications that do not do range-scans altogether. For these applications, a sorted set might not provide the optimal performance. For this reason, rocksdb supports a pluggable api that allows an application to provide their own implementation of a memtable. There are three memtables that are part of the library: a skiplist memtable, a vector memtable and a prefix-hash memtable. A vector memtable is appropriate for bulk-loading data into the database. Every write inserts a new element at the end of the vector; when it is time to flush the memtable to storage the elements in the vector are sorted and written out to a file in L0. A prefix-hash memtable allows efficient processing of gets, puts and scans-within-a-key-prefix.

### Memtable Pipelining:
Rocksdb supports configuring an arbitrary number of memtables for a database. When a memtable is full it becomes a immutable memtable and a background thread starts flushing its contents to storage. Meanwhile, new writes continue to accumulate to a newly allocated memtable. If the newly allocated memtable is filled up to its limit, it too is converted to a immutable memtable and inserted into the flush pipeline. The background thread continues to flush all the pipelined immutable memtables to storage. This pipelining increases write throughput of rocksdb especially when it is operating on slow storage devices.

### Memtable Compaction:
When a memtable is being flushed to storage, an inline-compaction process removes duplicate records from the output steam. Similarly, if an earlier put is hidden by a later delete, then the put is not written to the output file at all. This feature reduces the size of data on storage and write amplification greatly. This is an essential feature when rocksdb is used as a producer-consumer-queue, especially when the lifetime of an element in the queue is very short-lived.

## Merge Operator. Rockdb natively supports three types of records, a Put record, a Delete record and a Merge record. When a compaction process encounters a Merge record, it invokes a application-specified method called the Merge Operator. The Merge can combine multiple Put and Merge records into a single one. This is a powerful feature and allows applications that typically do read-modify-writes to completely avoid the reads. It allows an application to record the intent-of-the-operation as a Merge Record and the rocksdb compaction process lazily applies that intent to the original value. This feature is described in detail in [Merge Operator](https://github.com/facebook/rocksdb/wiki/Merge-Operator)

## Compaction Filter. Rocksdb allows an application to hook in application code into the rocksdb compaction process. For example, an application can continuously run a data sanitizer as part of the compaction. If the application wants to continuously delete data older than a specific time, then it can use the compaction filter to drop records that have expired.
 

### Author: Dhruba Borthakur
## Overview

Every update to RocksDB is written to two places: 1) an in-memory data structure called memtable (to be flushed to SST files later) and 2) write ahead log(WAL) on disk. In the event of a failure, write ahead logs can be used to completely recover the data in the memtable, which is necessary to restore the database to the original state. In the default configuration, RocksDB guarantees process crash consistency by fflush()ing the WAL after every user write.

## Life Cycle of a WAL

Let's use an example to illustrate the life cycle of a WAL. A RocksDB instace `db` is created with two [column families](https://github.com/facebook/rocksdb/wiki/Column-Families) "new_cf" and "default". Once the `db` is opened, a new WAL will be created on disk to persist all writes.
```
DB* db;
std::vector<ColumnFamilyDescriptor> column_families;
column_families.push_back(ColumnFamilyDescriptor(
    kDefaultColumnFamilyName, ColumnFamilyOptions()));
column_families.push_back(ColumnFamilyDescriptor(
    "new_cf", ColumnFamilyOptions()));
std::vector<ColumnFamilyHandle*> handles;
s = DB::Open(DBOptions(), kDBPath, column_families, &handles, &db);
```

Some key-value pairs are added to both column families

```
db->Put(WriteOptions(), handles[1], Slice("key1"), Slice("value1"));
db->Put(WriteOptions(), handles[0], Slice("key2"), Slice("value2"));
db->Put(WriteOptions(), handles[1], Slice("key3"), Slice("value3"));
db->Put(WriteOptions(), handles[0], Slice("key4"), Slice("value4"));
```

At this point the WAL should have recorded all writes. The WAL will stay open and keep recording future writes until its size reaches `DBOptions::max_total_wal_size`.

If user decides to flush the column family "new_cf", several things happens: 1) new_cf's data (key1 and key3) is flushed to a new SST file 2) a new WAL is created and all future writes to all column families now go to the new WAL 3) the older WAL will not accept new writes but the deletion may be delayed.

```
db->Flush(FlushOptions(), handles[1]);
// key5 and key6 will appear in a new WAL
db->Put(WriteOptions(), handles[1], Slice("key5"), Slice("value5"));
db->Put(WriteOptions(), handles[0], Slice("key6"), Slice("value6"));
```

At this point there will be two WALs, the older WAL contains key1 through key4 and newer WAL contains key5 and key6. Because the older WAL still contains live data for at least one column family ("default"), it cannot be deleted yet. Only when user finally decides to flush "default" column family, the older WAL can be archived and purged from disk automatically.

```
db->Flush(FlushOptions(), handles[0]);
// The older WAL will be archived and purged separetely
```

To summarize, a WAL is created when 1) a new DB is opened, 2) a column family is flushed. A WAL is deleted (or archived if archival is enabled) when all column families have flushed beyond the largest sequence number contained in the WAL, or in other words, all data in the WAL have been persisted to SST files. Archived WALs will be moved to a separete location and purged from disk later on. The actual deletion might be delayed due to replication purposes, see Transcational Log Iterator section below.

## WAL Configurations

The following configuration can be found in [options.h](https://github.com/facebook/rocksdb/blob/5.10.fb/include/rocksdb/options.h)

#### DBOptions::wal_dir

`DBOptions::wal_dir` sets the directory where RocksDB stores write-ahead log files, which allows WALs to be stored in a separate directory from the actual data.

#### DBOptions::WAL_ttl_seconds, DBOptions::WAL_size_limit_MB

These two fields affect how quickly archived WALs will be deleted. Nonzero values indicate the time and disk space threshold to trigger archived WAL deletion. See [options.h](https://github.com/facebook/rocksdb/blob/5.10.fb/include/rocksdb/options.h#L554-L565) for detailed explanation.

#### DBOptions::max_total_wal_size

In order to limit the size of WALs, RocksDB uses `DBOptions::max_total_wal_size` as the trigger of column family flush. Once WALs exceed this size, RocksDB will start forcing the flush of column families to allow deletion of some oldest WALs. This config can be useful when column families are updated at non-uniform frequencies. If there's no size limit, users may need to keep really old WALs when the infrequently-updated column families hasn't flushed for a while. 

#### DBOptions::avoid_flush_during_recovery

This config is self explanatory.

#### DBOptions::manual_wal_flush

`DBOptions::manual_wal_flush` determines whether WAL flush will be automatic after every write or purely manual (user must invoke `FlushWAL` to trigger a WAL flush).

#### DBOptions::wal_filter

Through `DBOptions::wal_filter`, users can provide a filter object to be invoked while processing WALs during recovery. 
_Note: Not supported in ROCKSDB_LITE mode_

#### WriteOptions::disableWAL

`WriteOptions::disableWAL` is useful when users rely on other logging or don't care about data loss.

## WAL Filter

## Transcation Log Iterator
Transaction log iterator provides a way to replicate the data between RocksDB instances. Once a WAL is archived due to column family flush, the WAL is archived instead of immediately deleted. The goal is to allow txn log iterator to keep reading the WAL and send to slave for replay. 

## Related Pages

#### WAL Recovery Modes
https://github.com/facebook/rocksdb/wiki/WAL-Recovery-Modes

#### WAL Log Format
https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format

**Q: If my process crashes, can it corrupt the database?**
A: No, but data in the un-flushed mem-tables might be lost if Write-Ahead-Log (WAL) is disabled.

**Q: If my machine crashes and rebooted, will RocksDB preserve the data?**
A: Data is synced when you issue a sync write (write with WriteOptions.sync=true), call DB::SyncWAL(), or when memtables are flushed.


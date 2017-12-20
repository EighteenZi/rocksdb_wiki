When users reopen a DB, here are some steps that may take longer than expected:

## Manifest replay
The [[MANIFEST]] file contains history of all file operations of the DB since the last time DB was opened, and is replayed during DB open. If there are too many updates to replay, it takes a long time. This can happen when:
* SST files were too small so file operations were too frequently. If this is the case, try to solve the small SST file problem. Maybe memtable is flushed too often, which generates small L0 files, or target file size is too small so that compaction generates small files. You can try to adjust the configuration accordingly
* DB simply runs for too long and accumulates too many historic updates.
Either way, you can try to set `options.max_manifest_file_size` to force a new manifest file to be generated when it hits the maximum size, to avoid replaying for too long.

## WAL replaying
All the WAL files are replayed if it contains any useful data.

If your memtable size is large, the replay can be long. So try to shrink the memtable size.

Another common reason that WAL files to replay is too large is that, one of the column families gets too slow writes, which holds logs from being deleted. When DB reopens, all those log files are read, just to replay updates from this column family. In this case, set a proper `options.max_total_wal_size` value. The low-traffic column families will be flushed to limit the total WAL files to replay to under this threshold. see [[Write Ahead Log]].

## Reading footer and meta blocks of all the SST files

## Opening too many DBs one by one
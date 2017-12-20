When users reopen a DB, here are some steps that may take longer than expected:

## Manifest replay
The [[MANIFEST]] file contains history of all file operations of the DB since the last time DB was opened, and is replayed during DB open. If there are too many updates to replay, it takes a long time. This can happen when:
* SST files were too small so file operations were too frequently. If this is the case, try to solve the small SST file problem. Maybe memtable is flushed too often, which generates small L0 files, or target file size is too small so that compaction generates small files. You can try to adjust the configuration accordingly
* DB simply runs for too long and accumulates too many historic updates.
Either way, you can try to set `options.max_manifest_file_size` to force a new manifest file to be generated when it hits the maximum size, to avoid replaying for too long.

## WAL replaying

## Reading footer and meta blocks of all the SST files

## Opening too many DBs one by one
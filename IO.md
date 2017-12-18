## Control Write I/O
### Range Sync
RocksDB's data files are usually generated in an appending way. File system may choose buffer the write until the dirty pages hit a threshold and write out all of those pages all together. This can create a burst of write I/O and cause the online I/Os to wait too long and cause long query latency. Rather, you can ask RocksDB to periodically hint OS to write out outstanding dirty pages by setting `options.bytes_per_sync` for SST files and `options.wal_bytes_per_sync` for WAL files. Underlying it calls sync_file_range() on Linux every time a file is appended for such size. The most recent pages are not included in the range sync.

### Rate Limiter
You can control the total rate RocksDB writes to data files through `options.rate_limiter`, in order to reserve enough I/O bandwidth to online queries. See [[Rate Limiter]] for details.

### Write Max Buffer
When appending a file, RocksDB has internal buffering of files before writing to the file system, unless an explicit fsync is needed. The max size of this buffer can be controlled by `options.writable_file_max_buffer_size`. Tuning this parameter is more critical in [[Direct IO] mode or to a file system without page cache. With non-direct I/O mode, enlarging this buffer only reduces number of write() system calls and is unlikely to change the I/O behavior, so unless this is what you want, it may be desirable to keep the default value 0 to save the memory.

## Control Read I/O
Coming soon.

## Direct I/O
Rather than control the I/O through file system hints shown above, you can enable direct I/O in RocksDB to allow RocksDB to directly control I/O, using option `use_direct_reads` and/or `use_direct_io_for_flush_and_compaction`. If direct I/O is enabled, some or all of the options introduced above will not be applicable. See more details in [[Direct IO]].
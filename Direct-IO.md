## Introduction
Direct I/O is a system-wide feature that supports direct reads/writes from/to storage device to/from user memory space bypassing system page cache. Buffered I/O is usually the default I/O mode enabled by most operating systems.

### Why do we need it?
With buffered I/O, the data is copied twice between storage and memory because of the page cache as the proxy between the two. In most cases, the introduction of page cache could achieve better performance. But for self-caching applications such as RocksDB, the application itself should have a better knowledge of the logical semantics of the data than OS, which provides a chance that the applications could implements more efficient replacement algorithm for cache with any application-defined data block as a unit by leveraging their knowledge of data semantics. On the other hand, in some situations, we want some data to opt-out of system cache. At this time, direct I/O would be a better choice.

### Implementation
The way to enable direct I/O depends on the operating system and the support of direct I/O depends on the file system. Before using this feature, please check whether the file system supports direct I/O. RocksDB has dealt with these OS-dependent complications for you, but we are glad to share some implementation details here.

1. File Open

   For LINUX, the `O_DIRECT` flag has to be included.
For Mac OSX, `O_DIRECT` is not available. Instead, `fcntl(fd, F_NOCACHE, 1)` looks to be the canonical solution where `fd` is the file descriptor of the file.
For Windows, there is a flag called `FILE_FLAG_NO_BUFFERING` as the counterpart in Windows of `O_DIRECT`.

2. File R/W

   Direct I/O requires file R/W to be aligned, which means, the position indicator (offset), #bytes and the buffer address must be aligned to the _logical sector size_ of the underlying storage device. So the position indicator should and the buffer pointer must be aligned on a _logical sector size_ boundary and the number of bytes to be read or written must be in multiples of the _logical sector size_.
RocksDB implements all the alignment logic inside `FileReader/FileWriter`, one layer higher abstraction on top of File classes to make the alignment ignorant to the OS. Thus, different OSs could have their own implementations of File Classes.

## API
It is easy to use Direct I/O as two new options are provided in [options.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h#L1124-L1128):
```cpp
// Use O_DIRECT for reading file
// Default: false
bool use_direct_reads = false;

// Use O_DIRECT for writing file
// Default: false
bool use_direct_writes = false;
```
The code is self-explanatory.

You may also need other options to optimize direct I/O performance.
```cpp
// options.h
// Option to enable readahead in compaction
size_t compaction_readahead_size = 2 * 1024 * 1024; // recommend at least 2MB
// Option to tune write buffer for direct writes
size_t writable_file_max_buffer_size = 1024 * 1024; // 1MB by default
```
```cpp
// table.h
BlockedBasedTableOptions bbto;
// For the users who want to cache compressed blocks by themselves,
// they can use compressed block cache
// Option to enable compressed block cache
bbto.block_cache_compressed = NewLRUCache(capacity);

// If true, block will not be explicitly flushed to disk during building
// a SstTable. Instead, buffer in WritableFileWriter will take
// care of the flushing when it is full.
bbto.skip_table_builder_flush = true;
```

###Notes 
1.  `allow_mmap_reads/use_direct_reads` and `allow_mmap_writes/use_direct_writes` are mutually exclusive, i.e., they cannot be set to true at the same time.
2.  Direct I/O options will only be applied to sst file I/O but not WAL I/O or MANIFEST I/O because the I/O pattern of these files are not suitable for direct I/O.
3. After enable direct I/O, compaction writes will no longer be in the OS page cache, so first read will do real IO. 
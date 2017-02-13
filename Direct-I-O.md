## Introduction
Direct I/O is a system-wide feature that supports direct reads/writes from/to storage device to/from user memory space bypassing system page cache. Buffered I/O is usually the default I/O mode enabled by most operating systems.

### Why do we need it?
With buffered I/O, the data is copied twice between storage and memory because of the page cache as the proxy between the two. In most cases, the introduction of page cache could achieve better performance. But for self-caching applications such as RocksDB, the application itself should have a better knowledge of the logical semantics of the data than OS, which provides a chance that the applications could implements more efficient replacement algorithm for cache with any application-defined data block as a unit by leveraging their knowledge of data semantics.
On the other hand, in some situations, we want some data to opt-out of system cache. At this time, direct I/O must be a better choice.

## API
It is easy to use Direct I/O as two new options are provided in [options.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h#L1124-L1128):
```
  // Use O_DIRECT for reading file
  // Default: false
  bool use_direct_reads = false;

  // Use O_DIRECT for writing file
  // Default: false
  bool use_direct_writes = false;
```
The code is self-explanatory.

###Notes 
1. `allow_mmap_reads/use_direct_reads` and `allow_mmap_writes/use_direct_writes` are mutually exclusive, i.e., they cannot be set to true at the same time.
2.  Direct I/O options will only be applied to sst file I/O but not WAL I/O or MANIFEST I/O because the I/O pattern of these files are not suitable for direct I/O.
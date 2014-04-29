Since RocksDB version 3.0, we started defining version macros in `include/rocksdb/version.h`:

    #define __ROCKSDB_MAJOR__ <major version>
    #define __ROCKSDB_MINOR__ <minor version>
    #define __ROCKSDB_PATCH__ <patch version>

That way, you can make your code compile and work with multiple versions of RocksDB, even though we change some of the API you might be using.
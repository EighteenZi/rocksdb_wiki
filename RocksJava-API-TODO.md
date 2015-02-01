This page sets out the known TODO items for the RocksJava API, it also shows who is thinking/working on a particular topic; Through this mechanism hopefully we can avoid duplicating effort.

## TODO

1. Fix locking issues with `Slice`/`DirectSlice` from `Comparator`. Move to thread-local storage of cached slice objects rather than single `Slice` locking with mutex.
**[@adamretter](https://github.com/adamretter)**

  1. Perhaps decide weather to use only `Slice` or `DirectSlice` to simplify overall API implementation, probably `DirectSlice` is more memory efficient.

  2. Use the Slice class in other Java classes where `byte[]` was previously used, so that we better align with C++ API, e.g. `RocksIterator`.

2. Rework `WBWIIterator` to use both `Slice` and `DirectSlice` (see above).

3. Implement ARM (Automatic Resource Management) e.g. `try-with-resources` Java 7 use via `Closeable`/`AutoCloseable` for iterators, db, write batches etc.

4. Introduce `final` on variables/members everywhere they are immutable.
**[@adamretter](https://github.com/adamretter)**

5. Implement `ldb` for Java. For example, the ability to specify the Comparator which implemented in Java.
**[@adamretter](https://github.com/adamretter)**

6. Custom merge operator for Java. At the moment it is only possible to use merge operators which are available in C++ but not implementing custom functionality solely in Java.
**[@fyrz](https://github.com/fyrz)**

7. Expose an AbstractLogger. RocksDB C++ api allows to provide a custom Logger. This shall also be possible from Java side and allows to attach RocksDB logging to application logging facilities.
**[@fyrz](https://github.com/fyrz)**

8. Port remaining functionality in `db.h` to RocksJava.
**[@fyrz](https://github.com/fyrz)**

9. Update Statistics/HistogramData to 3.10
**[@fyrz](https://github.com/fyrz)**

10. Build isolation. Building Java API should not require building RocksDB. You should be able to use a Java API build with a separate existing RocksDB installation. The Java API native aspect will instead indirectly depend on a shared or static RocksDB lib.
**[@adamretter](https://github.com/adamretter)**

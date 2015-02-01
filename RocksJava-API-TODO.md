This page sets out the known TODO items for the RocksJava API, it also shows who is thinking/working on a particular topic; Through this mechanism hopefully we can avoid duplicating effort.

## TODO

1. Fix locking issues with `Slice`/`DirectSlice` from `Comparator`. Move to thread-local storage of cached slice objects rather than single `Slice` locking with mutex.
**[@adamretter](https://github.com/adamretter)**

  1. Perhaps decide weather to use only `Slice` or `DirectSlice` to simplify overall API implementation, probably `DirectSlice` is more memory efficient.

  2. Use the Slice class in other Java classes where `byte[]` was previously used, so that we better align with C++ API, e.g. `RocksIterator`.

2. Rework WBWIIterator to use both `Slice` and `DirectSlice` (see above).

3. Implement ARM (Automatic Resource Management) e.g. `try-with-resources` Java 7 use via `Closeable`/`AutoCloseable` for iterators, db, write batches etc.

4. Introduce `final` on variables/members everywhere they are immutable.
**[@adamretter](https://github.com/adamretter)**

5. Implement `ldb` for Java. For example, the ability to specify the Comparator which implemented in Java.
**[@adamretter](https://github.com/adamretter)**
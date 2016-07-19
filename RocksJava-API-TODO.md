This page sets out the known TODO items for the RocksJava API, it also shows who is thinking/working on a particular topic; Through this mechanism hopefully we can avoid duplicating effort.

## Upcoming changes to the Java API

1. Adjust RocksJava Comparator implementation - We analyzed the current implementation and noticed a significant loss of performance using the current implementation. So we decided to do the following steps in order
  1. Analyze which one of the comparator implementations is performing better either `DirectComparator` or `Comparator`
  2. Outline a proper way to use Custom-C++-Comparators with RocksJava.
  3. Remove everything but one Comparator implementation. Depending on the analysis listed above.
  4. Document the performance penalties in related JavaDoc.
  5. `FindShortestSeparator`and `FindShortSuccessor` shall only do something if the Java method is implemented. What`s currently not the case.

2. Rework `WBWIIterator` to use both `Slice` and `DirectSlice` (see above).

3. ~~Introduce `final` on variables/members everywhere they are immutable.~~

4. Implement `ldb` for Java. For example, the ability to specify the Comparator which implemented in Java.
**[@adamretter](https://github.com/adamretter)**

5. ~~Custom merge operator for Java. At the moment it is only possible to use merge operators which are available in C++ but not implementing custom functionality solely in Java.~~ Decision: will not be implemented.

6. ~~Expose an AbstractLogger. RocksDB C++ api allows to provide a custom Logger. This shall also be possible from Java side and allows to attach RocksDB logging to application logging facilities.~~
  1. ~~Document the performance penalties if log level is too verbose.~~

7. Port remaining functionality in `db.h` to RocksJava.
**[@fyrz](https://github.com/fyrz)**

8. ~~Update Statistics/HistogramData to 3.10~~
**[@fyrz](https://github.com/fyrz)**

9. Build isolation. Building Java API should not require building RocksDB. You should be able to use a Java API build with a separate existing RocksDB installation. The Java API native aspect will instead indirectly depend on a shared or static RocksDB lib.
**[@adamretter](https://github.com/adamretter)**

10. Expose optimistic locking
**[@fyrz](https://github.com/fyrz)**

## Planned changes for 4.x

1. Move on to Java-8, especially because Java-7 is EOL this year.
  1. Look at Java 8 Long#unsigned operations.
  2. Consider whether we should add an UnsignedLong, UnsignedInt class types of our own to prevent users from sending invalid DBOptions.

2. Restructure the package layout within the Java part.

3. ~~Implement ARM (Automatic Resource Management) e.g. `try-with-resources` Java 7 use via `Closeable`/`AutoCloseable` for iterators, db, write batches etc. Along with this change we will remove the auto-cleanup for c++ resources using `finalize`. Instead we will throw an exception if a C++ resource is going to be finalized without freeing the native handle first.~~ **DONE [@adamretter](https://github.com/adamretter)**

4. Consider converting callbacks to lambda expressions
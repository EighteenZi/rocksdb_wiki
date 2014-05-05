RocksJava is a project to build high performance but easy-to-use Java driver for RocksDB.  In this page you will learn the basics of RocksDB Java API.

## Getting Started
To build RocksJava, the first thing is to specify the environmental variable JAVA_HOME to where you install Java SDK.  Once JAVA_HOME is properly set, simply run `make rocksdbjava` will build the Java binding for RocksDB:

```bash
$ make rocksdbjava
```

This will generate `rocksdbjni.jar` and `librocksdbjni.so` (or librocksdbjni.jnilib in Mac) in `java/` directory under the rocksdb's root directory.  Specifically, `rocksdbjni.jar` contains the Java classes that defines the Java API for RocksDB, while `librocksdbjni.so` includes the C++ rocksdb library and the native implementation of the Java classes defined in `rocksdbjni.jar`.

Once `make rocksdbjava` is completed, `make jtest` will build and run the RocksJava sample code and tests.  The output should look something like the followings:

```bash
$ make jtest
javac org/rocksdb/util/*.java org/rocksdb/*.java
jar -cf rocksdbjni.jar org/rocksdb/*.class org/rocksdb/util/*.class
javah -d ./include -jni org.rocksdb.RocksDB org.rocksdb.Options org.rocksdb.WriteBatch org.rocksdb.WriteBatchInternal org.rocksdb.WriteBatchTest org.rocksdb.WriteOptions org.rocksdb.BackupableDB org.rocksdb.BackupableDBOptions org.rocksdb.Statistics org.rocksdb.Iterator org.rocksdb.VectorMemTableConfig org.rocksdb.SkipListMemTableConfig org.rocksdb.HashLinkedListMemTableConfig org.rocksdb.HashSkipListMemTableConfig org.rocksdb.PlainTableConfig org.rocksdb.ReadOptions org.rocksdb.Filter org.rocksdb.BloomFilter
javac -cp rocksdbjni.jar RocksDBSample.java
java -ea -Djava.library.path=.:../ -cp ".:./*" -Xcheck:jni RocksDBSample /tmp/rocksdbjni
RocksDBSample
caught the expceted exception -- org.rocksdb.RocksDBException: Invalid argument: /tmp/rocksdbjni_not_found: does not exist (create_if_missing is false)
Get('hello') = world
1 2 3 4 5 6 7 8 9
2 4 6 8 10 12 14 16 18
3 6 9 12 15 18 21 24 27
4 8 12 16 20 24 28 32 36
5 10 15 20 25 30 35 40 45
6 12 18 24 30 36 42 48 54
7 14 21 28 35 42 49 56 63
8 16 24 32 40 48 56 64 72
9 18 27 36 45 54 63 72 81
getTickerCount() passed.
geHistogramData() passed.
iterator seekToFirst tests passed.
iterator seekToLastPassed tests passed.
iterator seek test passed.
iterator tests passed.
javac org/rocksdb/util/*.java org/rocksdb/*.java
jar -cf rocksdbjni.jar org/rocksdb/*.class org/rocksdb/util/*.class
javah -d ./include -jni org.rocksdb.RocksDB org.rocksdb.Options org.rocksdb.WriteBatch org.rocksdb.WriteBatchInternal org.rocksdb.WriteBatchTest org.rocksdb.WriteOptions org.rocksdb.BackupableDB org.rocksdb.BackupableDBOptions org.rocksdb.Statistics org.rocksdb.Iterator org.rocksdb.VectorMemTableConfig org.rocksdb.SkipListMemTableConfig org.rocksdb.HashLinkedListMemTableConfig org.rocksdb.HashSkipListMemTableConfig org.rocksdb.PlainTableConfig org.rocksdb.ReadOptions org.rocksdb.Filter org.rocksdb.BloomFilter
javac org/rocksdb/test/*.java
java -ea -Djava.library.path=.:../ -cp "rocksdbjni.jar:.:./*" org.rocksdb.WriteBatchTest
Testing WriteBatchTest.Empty ===
Testing WriteBatchTest.Multiple ===
Testing WriteBatchTest.Append ===
Testing WriteBatchTest.Blob ===
Passed all WriteBatchTest!
java -ea -Djava.library.path=.:../ -cp "rocksdbjni.jar:.:./*" org.rocksdb.test.BackupableDBTest
java -ea -Djava.library.path=.:../ -cp "rocksdbjni.jar:.:./*" org.rocksdb.test.OptionsTest
Passed OptionsTest
java -ea -Djava.library.path=.:../ -cp "rocksdbjni.jar:.:./*" org.rocksdb.test.ReadOptionsTest
Passed ReadOptionsTest
```

## Opening a Database
A `rocksdb` database has a name which corresponds to a file system directory. All of the contents of database are stored in this directory. The following example shows how to open a database, creating it if necessary:

```java
import org.rocksdb.RocksDB;
import org.rocksdb.Options;
...
  // a static method that loads the RocksDB C++ library.
  RocksDB.loadLibrary();
  // the Options class contains a set of configurable DB options
  // that determines the behavior of a database.
  Options options = new Options().setCreateIfMissing(true);
  RocksDB db = null;
  try {
    // a factory method that returns a RocksDB instance
    db = RocksDB.open(options, "path/to/db");
    // do something
  } catch (RocksDBException e) {
    // do some error handling
    ...
  }
...
```

> TIP: You may notice the `RocksDBException` class above.  This exception class extends `java.lang.Exception` and encapsulates the `Status` class in the C++ rocksdb, which describes any internal errors of RocksDB.

## Closing a Database
When you are done with a database, it is suggested to call `RocksDB.close()` (or its equivalent method `dispose()`) and `options.dispose()` to release their C++ resource manually, although these associated C++ resources will also be released in their finalizer:

```java
  if (db != null) db.close();
  options.dispose();
```

> **TIP**: When you see a class extends `RocksObject`, it means any instance of this class has a native handle which stores a C++ pointer pointing to some rocksdb related resource.  It is suggested to invoke `RocksObject.dispose()` manually once all its associated databases have been closed.  Note that calling a non-static method of an already-disposed RocksObject instance is an undefined behavior.

## Reads and Writes
The database provides `put`, `remove`, and `get` methods to modify/query the database. For example, the following code moves the value stored under `key1` to `key2`.

```java
byte[] key1;
byte[] key2;
// some initialization for key1 and key2
try {
  byte[] value = db.get(key1);
  if (value != null) {  // value == null if key1 does not exist in db.
    db.put(key2, value);
  }
  db.remove(key1);
} catch (RocksDBException e) {
  // error handling
}
```

> **TIP**: You can also control the `put` and `get` behavior using `WriteOptions` and `ReadOptions` by calling their polymorphic methods `RocksDB.put(WriteOptions opt, byte[] key, byte[] value)` and `RocksDB.get(ReadOptions opt, byte[] key)`.

<!-- separator -->
> **TIP**: To avoid creating a byte-array in `RocksDB.get()`, you can also use its parametric method `int RocksDB.get(byte[] key, byte[] value)` or `int RocksDB.get(ReadOptions opt, byte[] key, byte[] value)`, where the output value will be filled into the pre-allocated output buffer `value`, and its `int` returned value will indicate the actual length of the value associated with the input `key`.  When the returned value is greater than `value.length`, this indicates the size of the output buffer is insufficient.

## Further Documentation
TBD
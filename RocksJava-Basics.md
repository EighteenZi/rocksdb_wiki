RocksJava is a project to build high performance but easy-to-use Java driver for RocksDB. 

(We try hard to keep RocksJava API in sync with RocksDB's C++ API, but it often falls behind. We highly encourage community contributions ... so please feel free to send us a Pull Request if you find yourself needing a certain API which is in C++ but not yet in Java.)

In this page you will learn the basics of RocksDB Java API.

## Getting Started
To build RocksJava, you first need to set your JAVA_HOME environment variable to point to the location where Java SDK is installed. Once JAVA_HOME is properly set, simply running `make rocksdbjava` will build the Java bindings for RocksDB:

```bash
$ make rocksdbjava
```

This will generate `rocksdbjni.jar` and `librocksdbjni.so` (or librocksdbjni.jnilib in Mac) in `java/target` directory under the rocksdb's root directory.  Specifically, `rocksdbjni.jar` contains the Java classes that defines the Java API for RocksDB, while `librocksdbjni.so` includes the C++ rocksdb library and the native implementation of the Java classes defined in `rocksdbjni.jar`.

To run unit tests:
```
$ make jtest
```
To clean:
```bash
$ make jclean
```

We also publish the JNI jars to maven, in case you just want to depend on the jar instead of building it on your own. 

## Samples
We provided some samples [here](https://github.com/facebook/rocksdb/tree/master/java/samples/src/main/java), if you want to jump directly into code.

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

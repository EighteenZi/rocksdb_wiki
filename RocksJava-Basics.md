RocksJava is a project to build high performance but easy-to-use Java driver for RocksDB. 

RocksJava is structured in 3 layers:

1. The Java classes within the `org.rocksdb` package which form the RocksJava API. Java users only directly interact with this layer.

2. JNI code written in C++ that provides the link between the Java API and RocksDB.

3. RocksDB itself written in C++ and compiled into a native library which is used by the JNI layer.

(We try hard to keep RocksJava API in sync with RocksDB's C++ API, but it often falls behind. We highly encourage community contributions ... so please feel free to send us a Pull Request if you find yourself needing a certain API which is in C++ but not yet in Java.)

In this page you will learn the basics of RocksDB Java API.

## Getting Started

You may either use the pre-built Maven artifacts that we publish, or build RocksJava yourself from its source code.

### Maven

We also publish RocksJava artifacts to Maven Central, in case you just want to depend on the jar instead of building it on your own: https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.rocksdb%22.

We publish both an uber-like Jar (rocksdbjni-X.X.X.jar) which contains native libraries for all supported platforms alongside the Java class files, as well as smaller platform specific Jars (such as rocksdbjni-X.X.X-linux64.jar).

The simplest way to use RocksJava from a build system which supports Maven style dependencies is to add a dependency on RocksJava. For example, if you are using Maven:

```xml
<dependency>
  <groupId>org.rocksdb</groupId>
  <artifactId>rocksdbjni</artifactId>
  <version>5.5.1</version>
</dependency>
```

### Compiling from Source
To build RocksJava, you first need to set your `JAVA_HOME` environment variable to point to the location where Java SDK is installed (must be Java 1.7+). You must also have the prerequisites for your platform to compile the native library of RocksDB, see [INSTALL.md](https://github.com/facebook/rocksdb/blob/master/INSTALL.md). Once `JAVA_HOME` is properly set and you have the prerequisites installed, simply running `make rocksdbjava` will build the Java bindings for RocksDB:

```bash
$ make -j8 rocksdbjava
```

This will generate `rocksdbjni.jar` and `librocksdbjni.so` (or librocksdbjni.jnilib on macOS) in the `java/target` directory under the rocksdb root directory. Specifically, `rocksdbjni.jar` contains the Java classes that defines the Java API for RocksDB, while `librocksdbjni.so` includes the C++ rocksdb library and the native implementation of the Java classes defined in `rocksdbjni.jar`.

To run the unit tests:
```bash
$ make jtest

```
To clean:
```bash
$ make jclean
```

## Samples
We provided some samples [here](https://github.com/facebook/rocksdb/tree/master/java/samples/src/main/java), if you want to jump directly into code.

## Memory Management
Many of the Java Objects used in the RocksJava API will be backed by C++ objects for which the Java Objects have ownership. As C++ has no notion of automatic garbage collection for its heap in the way that Java does, we must explicitly free the memory used by the C++ objects when we are finished with them.

Any Java object in RocksJava that manages a C++ object will inherit from `org.rocksdb.AbstractNativeReference` which is designed to assist in managing and cleaning up any owned C++ objects when you are done with them. Two mechanisms are used for this:

1. `AbstractNativeReference#close()`.

    This method should be explicitly invoked by the user when they have finished with a RocksJava object. If C++ objects were allocated and have not yet been freed then they will be released on the first invocation of this method.

    To ease the use of this, this method overrides `java.lang.AutoCloseable#close()`, which enables it to be used with ARM (Automatic Resource Management) like constructs such as Java SE 7's [`try-with-resources`](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html) statement.

2. `AbstractNativeReference#finalize()`.

    This method is called by Java's Finalizer thread, when all strong references to the object have expired and just before the object is Garabage Collected. Ultimately it delegates to `AbstractNativeReference#close()`. The user should however not rely on this, and instead consider it more of a last-effort fail-safe.

    It will certainly make sure that owned C++ objects will be cleaned up when the Java object is collected. It does not however help to manage the memory of RocksJava as a whole, as the memory allocated on the heap in C++ for the native C++ objects backing the Java objects is effectively invisible to the Java GC process and so the JVM cannot correctly calculate the memory pressure for the GC. **Users should always explicitly call `AbstractNativeReference#close()`** on their RocksJava objects when they are done with them.

## Opening a Database
A `rocksdb` database has a name which corresponds to a file system directory. All of the contents of database are stored in this directory. The following example shows how to open a database, creating it if necessary:

```java
import org.rocksdb.RocksDB;
import org.rocksdb.Options;
...
  // a static method that loads the RocksDB C++ library.
  RocksDB.loadLibrary();

  // the Options class contains a set of configurable DB options
  // that determines the behaviour of the database.
  try (final Options options = new Options().setCreateIfMissing(true)) {
    
    // a factory method that returns a RocksDB instance
    try (final RocksDB db = RocksDB.open(options, "path/to/db")) {
    
        // do something
    }
  } catch (RocksDBException e) {
    // do some error handling
    ...
  }
...
```

> TIP: You may notice the `RocksDBException` class above.  This exception class extends `java.lang.Exception` and encapsulates the `Status` class in the C++ rocksdb, which describes any internal errors of RocksDB.

## Reads and Writes
The database provides `put`, `remove`, and `get` methods to modify/query the database. For example, the following code moves the value stored under `key1` to `key2`.

```java
byte[] key1;
byte[] key2;
// some initialization for key1 and key2

try {
  final byte[] value = db.get(key1);
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

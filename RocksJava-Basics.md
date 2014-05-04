RocksJava is a project to build high performance but easy-to-use Java driver for RocksDB.  In this page you will learn the basics of RocksDB Java API.

# Getting Started
To build RocksJava, the first thing is to specify the environmental variable JAVA_HOME to where you install Java SDK.  Once JAVA_HOME is properly set, simply run `make rocksdbjava` will build the Java binding for RocksDB:

```bash
rocksdb $ make rocksdbjava
```

This will generate `rocksdbjni.jar` and `librocksdbjni.so` (or librocksdbjni.jnilib in Mac) in `java/` directory under the rocksdb's root directory.  Specifically, `rocksdbjni.jar` contains the Java classes that defines the Java API for RocksDB, while `librocksdbjni.so` includes the C++ rocksdb library and the native implementation of the Java classes defined in `rocksdbjni.jar`.

Once `make rocksdbjava` is completed, `make jtest` will build and run the RocksJava sample code and tests.  The output should look something like the followings:

```bash
rocksdb $ make jtest
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



RocksJava is a project that we launched in April 2014 to build a high performance Java driver for RocksDB.  Here we would like to show its performance numbers on flash storage.  We will first show the result summary.  Details about experimental setup and commands to run the benchmark will be covered in the later sections.

# Result Summary
We repeated the benchmarks on flash storage described in [3] and compare the performance between RocksJava and the RocksDB C++.  In this benchmark, the database has one billion key-values, and each key / value has 16 / 800 bytes respectively.  Below shows the summary of the results:

### Table 1.  Performance comparison over 1TB database on flash storage.
| Benchmark                          | RocksJava |RocksDB C++| Diff (%)| Details |
|------------------------------------|----------:|------------:|:-------:|:--------|
|1 Sequential Writer                 | 369k wps  |  371K wps   |   < 1%  | [Seq Bulk Load](#fillseq) |
|32 Random Readers	             | 270K rps  |  303K rps   | -10.8%  |         |
|32 Random Readers <br/>w/ 1 Random Writer| 206K rps  |  336K rps   | -38.5%  |         |
|32 Sequential Readers               |2.12M rps  | 6.84M rps   | -69.0%  |         |
<br/>

Here we further discuss the 32 sequential readers benchmark where RocksJava is 70% slower than RocksDB C++.

Sequential-read is a relative high qps operation compared to the operations in other benchmarks. As a result, the overhead on the Java side will become more noticeable. In addition, in the current implementation, each read in RocksJava's sequential reader involves in four JNI calls: which are `next()`, `isValid()`, `key()`, and `value()`, while the operations used in other benchmarks only involve one JNI call:

```java
    @Override public void runTask() throws RocksDBException {
      org.rocksdb.Iterator iter = db_.newIterator();
      long i;
      for (iter.seekToFirst(), i = 0;
           iter.isValid() && i < numEntries_;
           iter.next(), ++i) {
        stats_.found_++;
        stats_.finishedSingleOp(iter.key().length + iter.value().length);
        if (isFinished()) {
          return;
        }
      }
    }
```

This can be improved by introducing a better api that combines some of these functions together (such as `boolean nextValid()`).

# Setup
We tried to reuse the settings used in [].  Here are some of important settings / difference used in our benchmark:

* Intel(R) Xeon(R) CPU E5-2660 v2 @ 2.20GHz, 40 cores.
* 25 MB CPU cache, 144 GB Ram
* CentOS release 5.2 (Final)
* Java version "1.7.0_55" (Java(TM) SE Runtime Environment (build 1.7.0_55-b13), Java HotSpot(TM) 64-Bit * * Server VM (build 24.55-b03, mixed mode)
* Test with 1 billion key / value pairs. Each key is 16 bytes, and each value is 800 bytes.
* Snappy compression is used.
* For 32 readers w/ 1 writer benchmark, the writer performs 10k writes per second.
* Does not use JEMALLOC.

## Bulk Load of keys in Sequential Order
<a name="fillseq"/>

## Random Read performance
<a name="readrandom"/>

## Multi-Threaded Random Read and Single-Threaded Write Performance
<a name="readwhilewriting/>

## Sequential Read Performance
<a name="readseq"/>
# Basic operations

The <code>rocksdb</code> library provides a persistent key value store. Keys and values are arbitrary byte arrays. The keys are ordered within the key value store according to a user-specified comparator function.


## Opening A Database

A <code>rocksdb</code> database has a name which corresponds to a file system directory. All of the contents of database are stored in this directory. The following example shows how to open a database, creating it if necessary:

```cpp
  #include <cassert>
  #include "rocksdb/db.h"

  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  assert(status.ok());
  ...
```

If you want to raise an error if the database already exists, add the following line before the <code>rocksdb::DB::Open</code> call:

```cpp
  options.error_if_exists = true;
```

If you are porting code from <code>leveldb</code> to <code>rocksdb</code>, you can convert your <code>leveldb::Options</code> object to a <code>rocksdb::Options</code> object using <code>rocksdb::LevelDBOptions</code>, which has the same functionality as <code>leveldb::Options</code>:

```cpp
  #include "rocksdb/utilities/leveldb_options.h"

  rocksdb::LevelDBOptions leveldb_options;
  leveldb_options.option1 = value1;
  leveldb_options.option2 = value2;
  ...
  rocksdb::Options options = rocksdb::ConvertOptions(leveldb_options);
```


## Status

You may have noticed the <code>rocksdb::Status</code> type above. Values of this type are returned by most functions in <code>rocksdb</code> that may encounter an error. You can check if such a result is ok, and also print an associated error message:

```cpp
   rocksdb::Status s = ...;
   if (!s.ok()) cerr << s.ToString() << endl;
```


## Closing A Database

When you are done with a database, just delete the database object.

Example:

```cpp
  ... open the db as described above ...
  ... do something with db ...
  delete db;
```


## Reads And Writes

The database provides <code>Put</code>, <code>Delete</code>, and <code>Get</code> methods to modify/query the database. For example, the following code moves the value stored under key1 to key2.

```cpp
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) s = db->Put(rocksdb::WriteOptions(), key2, value);
  if (s.ok()) s = db->Delete(rocksdb::WriteOptions(), key1);
```


## Atomic Updates

Note that if the process dies after the Put of key2 but before the delete of key1, the same value may be left stored under multiple keys. Such problems can be avoided by using the <code>WriteBatch</code> class to atomically apply a set of updates:

```cpp
  #include "rocksdb/write_batch.h"
  ...
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) {
    rocksdb::WriteBatch batch;
    batch.Delete(key1);
    batch.Put(key2, value);
    s = db->Write(rocksdb::WriteOptions(), &batch);
  }
```

The <code>WriteBatch</code> holds a sequence of edits to be made to the database, and these edits within the batch are applied in order. Note that we called <code>Delete</code> before <code>Put</code> so that if <code>key1</code> is identical to <code>key2</code>, we do not end up erroneously dropping the value entirely.

Apart from its atomicity benefits, <code>WriteBatch</code> may also be used to speed up bulk updates by placing lots of individual mutations into the
same batch.


## Synchronous Writes
 
By default, each write to <code>leveldb</code> is asynchronous: it returns after pushing the write from the process into the operating system. The transfer from operating system memory to the underlying persistent storage happens asynchronously. The <code>sync</code> flag can be turned on for a particular write to make the write operation not return until the data being written has been pushed all the way to persistent storage. (On Posix systems, this is implemented by calling either <code>fsync(...)</code> or <code>fdatasync(...)</code> or <code>msync(..., MS_SYNC)</code> before the write operation returns.) 

```cpp
  rocksdb::WriteOptions write_options;
  write_options.sync = true;
  db->Put(write_options, ...);
```


## Asynchronous Writes 

Asynchronous writes are often more than a thousand times as fast as synchronous writes. The downside of asynchronous writes is that a crash of the machine may cause the last few updates to be lost. Note that a crash of just the writing process (i.e., not a reboot) will not cause any loss since even when <code>sync</code> is false, an update is pushed from the process memory into the operating system before it is considered done.

Asynchronous writes can often be used safely. For example, when loading a large amount of data into the database you can handle lost updates by restarting the bulk load after a crash. A hybrid scheme is also possible where every Nth write is synchronous, and in the event of a crash, the bulk load is restarted just after the last synchronous write finished by the previous run. (The synchronous write can update a marker that describes where to restart on a crash.)


<code>WriteBatch</code> provides an alternative to asynchronous writes. Multiple updates may be placed in the same <code>WriteBatch</code> and applied together using a synchronous write (i.e., <code>write_options.sync</code> is set to true). The extra cost of the synchronous write will be amortized across all of the writes in the batch.


We also provide a way to completely disable Write Ahead Log for a particular write. If you set write_option.disableWAL to true, the write will not go to the log at all and may be lost in an event of process crash.

When opening a DB, you can disable syncing of data files by setting Options::disableDataSync to true. This can be useful when doing bulk-loading or big idempotent operations. Once the operation is finished, you can manually call sync() to flush all dirty buffers to stable storage.

RocksDB by default uses faster fdatasync() to sync files. If you want to use fsync(), you can set Options::use_fsync to true. You should set this to true on filesystems like ext3 that can lose files after a reboot.


## Concurrency

A database may only be opened by one process at a time. The <code>rocksdb</code> implementation acquires a lock from the operating system to prevent misuse. Within a single process, the same <code>rocksdb::DB</code> object may be safely shared by multiple concurrent threads. I.e., different threads may write into or fetch iterators or call <code>Get</code> on the same database without any external synchronization (the leveldb implementation will automatically do the required synchronization). However other objects (like Iterator and WriteBatch) may require external synchronization. If two threads share such an object, they must protect access to it using their own locking protocol. More details are available in the public header files.


## Merge operators

Merge operators provide efficient support for read-modify-write operation.
More on the interface and implementation can be found on:
* [[Merge Operator | Merge-Operator]]
* [[Merge Operator Implementation | Merge-Operator-Implementation]]


## Iteration

The following example demonstrates how to print all key,value pairs in a database.

```cpp
  rocksdb::Iterator* it = db->NewIterator(rocksdb::ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    cout << it->key().ToString() << ": " << it->value().ToString() << endl;
  }
  assert(it->status().ok()); // Check for any errors found during the scan
  delete it;
```
The following variation shows how to process just the keys in the
range <code>[start,limit)</code>:

```cpp
  for (it->Seek(start);
       it->Valid() && it->key().ToString() < limit;
       it->Next()) {
    ...
  }
```
You can also process entries in reverse order. (Caveat: reverse
iteration may be somewhat slower than forward iteration.)

```cpp
  for (it->SeekToLast(); it->Valid(); it->Prev()) {
    ...
  }
```


## Snapshots

Snapshots provide consistent read-only views over the entire state of the key-value store. <code>ReadOptions::snapshot</code> may be non-NULL to indicate that a read should operate on a particular version of the DB state. 

If <code>ReadOptions::snapshot</code> is NULL, the read will operate on an implicit snapshot of the current state.

Snapshots are created by the DB::GetSnapshot() method:

```cpp
  rocksdb::ReadOptions options;
  options.snapshot = db->GetSnapshot();
  ... apply some updates to db ...
  rocksdb::Iterator* iter = db->NewIterator(options);
  ... read using iter to view the state when the snapshot was created ...
  delete iter;
  db->ReleaseSnapshot(options.snapshot);
```

Note that when a snapshot is no longer needed, it should be released using the DB::ReleaseSnapshot interface. This allows the implementation to get rid of state that was being maintained just to support reading as of that snapshot.


## Slice

The return value of the <code>it->key()</code> and <code>it->value()</code> calls above are instances of the <code>rocksdb::Slice</code> type. <code>Slice</code> is a simple structure that contains a length and a pointer to an external byte array. Returning a <code>Slice</code> is a cheaper alternative to returning a <code>std::string</code> since we do not need to copy potentially large keys and values. In addition, <code>rocksdb</code> methods do not return null-terminated C-style strings since <code>rocksdb</code> keys and values are allowed to contain '\0' bytes.

C++ strings and null-terminated C-style strings can be easily converted to a Slice:

```cpp
   rocksdb::Slice s1 = "hello";

   std::string str("world");
   rocksdb::Slice s2 = str;
```

A Slice can be easily converted back to a C++ string:

```cpp
   std::string str = s1.ToString();
   assert(str == std::string("hello"));
```

Be careful when using Slices since it is up to the caller to ensure that the external byte array into which the Slice points remains live while the Slice is in use. For example, the following is buggy:

```cpp
   rocksdb::Slice slice;
   if (...) {
     std::string str = ...;
     slice = str;
   }
   Use(slice);
```

When the <code>if</code> statement goes out of scope, <code>str</code> will be destroyed and the backing storage for <code>slice</code> will disappear.


## Comparators

The preceding examples used the default ordering function for key, which orders bytes lexicographically. You can however supply a custom comparator when opening a database. For example, suppose each database key consists of two numbers and we should sort by the firstnumber, breaking ties by the second number. First, define a proper subclass of <code>rocksdb::Comparator</code> that expresses these rules:

```cpp
  class TwoPartComparator : public rocksdb::Comparator {
   public:
    // Three-way comparison function:
    // if a < b: negative result
    // if a > b: positive result
    // else: zero result
    int Compare(const rocksdb::Slice& a, const rocksdb::Slice& b) const {
      int a1, a2, b1, b2;
      ParseKey(a, &a1, &a2);
      ParseKey(b, &b1, &b2);
      if (a1 < b1) return -1;
      if (a1 > b1) return +1;
      if (a2 < b2) return -1;
      if (a2 > b2) return +1;
      return 0;
    }

    // Ignore the following methods for now:
    const char* Name() const { return "TwoPartComparator"; }
    void FindShortestSeparator(std::string*, const rocksdb::Slice&) const { }
    void FindShortSuccessor(std::string*) const { }
  };
```

Now create a database using this custom comparator:

```cpp
  TwoPartComparator cmp;
  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  options.comparator = &cmp;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  ...
```


## Backwards compatibility

The result of the comparator's <code>Name</code> method is attached to the database when it is created, and is checked on every subsequent database open. If the name changes, the <code>rocksdb::DB::Open</code> call will fail. Therefore, change the name if and only if the new key format and comparison function are incompatible with existing databases, and it is ok to discard the contents of all existing databases.

You can however still gradually evolve your key format over time with a little bit of pre-planning. For example, you could store a version number at the end of each key (one byte should suffice for most uses). When you wish to switch to a new key format (e.g., adding an optional third part to the keys processed by <code>TwoPartComparator</code>), (a) keep the same comparator name (b) increment the version number for new keys (c) change the comparator function so it uses the version numbers found in the keys to decide how to interpret them.



## MemTable and Table factories

By default, we keep the data in memory in skiplist memtable and the data on disk in a table format described here: <a href="https://github.com/facebook/rocksdb/wiki/Rocksdb-Table-Format">RocksDB Table Format</a>.

Since one of the goals of RocksDB is to have different parts of the system easily pluggable, we support different implementations of both memtable and table format. You can supply your own memtable factory by setting <code>Options::memtable_factory</code> and your own table factory by setting <code>Options::table_factory</code>. For available memtable factories, please refer to <code>rocksdb/memtablerep.h</code> and for table factores to <code>rocksdb/table.h</code>. These features are both in active development and please be wary of any API changes that might break your application going forward.

You can also read more about memtables [here](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics#memtables).

## Performance

Performance can be tuned by changing the default values of the types defined in <code>include/rocksdb/options.h</code>.


## Block size

<code>rocksdb</code> groups adjacent keys together into the same block and such a block is the unit of transfer to and from persistent storage. The default block size is approximately 4096 uncompressed bytes. Applications that mostly do bulk scans over the contents of the database may wish to increase this size. Applications that do a lot of point reads of small values may wish to switch to a smaller block size if performance measurements indicate an improvement. There isn't much benefit in using blocks smaller than one kilobyte, or larger than a few megabytes. Also note that compression will be more effective with larger block sizes. To change block size parameter, use <code>Options::block_size</code>.

## Write buffer

<code>Options::write_buffer_size</code> specifies the amount of data to build up in memory before converting to a sorted on-disk file. Larger values increase performance, especially during bulk loads. Up to max_write_buffer_number write buffers may be held in memory at the same time, so you may wish to adjust this parameter to control memory usage. Also, a larger write buffer will result in a longer recovery time the next time the database is opened.

Related option is <code>Options::max_write_buffer_number</code>, which is maximum number of write buffers that are built up in memory. The default is 2, so that when 1 write buffer is being flushed to storage, new writes can continue to the other write buffer.

<code>Options::min_write_buffer_number_to_merge</code> is the minimum number of write buffers that will be merged together before writing to storage. If set to 1, then all write buffers are flushed to L0 as individual files and this increases read amplification because a get request has to check in all of these files. Also, an in-memory merge may result in writing lesser data to storage if there are duplicate records in each of these individual write buffers. Default: 1

## Compression

Each block is individually compressed before being written to persistent storage. Compression is on by default since the default compression method is very fast, and is automatically disabled for uncompressible data. In rare cases, applications may want to disable compression entirely, but should only do so if benchmarks show a performance improvement:

```cpp
  rocksdb::Options options;
  options.compression = rocksdb::kNoCompression;
  ... rocksdb::DB::Open(options, name, ...) ....
```


## Cache

The contents of the database are stored in a set of files in the filesystem and each file stores a sequence of compressed blocks. If <code>options.block_cache</code> is non-NULL, it is used to cache frequently used uncompressed block contents. We use operating systems file cache to cache our raw data, which is compressed. So file cache acts as a cache for compressed data.

```cpp
  #include "rocksdb/cache.h"
  rocksdb::BlockBasedTableOptions table_options;
  table_options.block_cache = rocksdb::NewLRUCache(100 * 1048576); // 100MB uncompressed cache

  rocksdb::Options options;
  options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));
  rocksdb::DB* db;
  rocksdb::DB::Open(options, name, &db);
  ... use the db ...
  delete db
```

When performing a bulk read, the application may wish to disable caching so that the data processed by the bulk read does not end up displacing most of the cached contents. A per-iterator option can be used to achieve this:

```cpp
  rocksdb::ReadOptions options;
  options.fill_cache = false;
  rocksdb::Iterator* it = db->NewIterator(options);
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    ...
  }
```

You can also disable block cache by setting <code>options.no_block_cache</code> to true.


## Key Layout

Note that the unit of disk transfer and caching is a block. Adjacent keys (according to the database sort order) will usually be placed in the same block. Therefore the application can improve its performance by placing keys that are accessed together near each other and placing infrequently used keys in a separate region of the key space.

For example, suppose we are implementing a simple file system on top of <code>rocksdb</code>. The types of entries we might wish to store are:

```cpp
   filename -> permission-bits, length, list of file_block_ids
   file_block_id -> data
```

We might want to prefix <code>filename</code> keys with one letter (say '/') and the <code>file_block_id</code> keys with a different letter (say '0') so that scans over just the metadata do not force us to fetch and cache bulky file contents.


## Filters

Because of the way <code>rocksdb</code> data is organized on disk, a single <code>Get()</code> call may involve multiple reads from disk. The optional <code>FilterPolicy</code> mechanism can be used to reduce the number of disk reads substantially.

```cpp
   rocksdb::Options options;
   options.filter_policy = NewBloomFilter(10);
   rocksdb::DB* db;
   rocksdb::DB::Open(options, "/tmp/testdb", &db);
   ... use the database ...
   delete db;
   delete options.filter_policy;
```

The preceding code associates a [[Bloom Filter | RocksDB-Bloom-Filter]] based filtering policy with the database. Bloom filter based filtering relies on keeping some number of bits of data in memory per key (in this case 10 bits per key since that is the argument we passed to NewBloomFilter). This filter will reduce the number of unnecessary disk reads needed for <code>Get()</code> calls by a factor of approximately a 100. Increasing the bits per key will lead to a larger reduction at the cost of more memory usage. We recommend that applications whose working set does not fit in memory and that do a lot of random reads set a filter policy.

If you are using a custom comparator, you should ensure that the filter policy you are using is compatible with your comparator. For example, consider a comparator that ignores trailing spaces when comparing keys. <code>NewBloomFilter</code> must not be used with such a comparator. Instead, the application should provide a custom filter policy that also ignores trailing spaces. 

For example:

```cpp
  class CustomFilterPolicy : public rocksdb::FilterPolicy {
   private:
    FilterPolicy* builtin_policy_;
   public:
    CustomFilterPolicy() : builtin_policy_(NewBloomFilter(10)) { }
    ~CustomFilterPolicy() { delete builtin_policy_; }

    const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

    void CreateFilter(const Slice* keys, int n, std::string* dst) const {
      // Use builtin bloom filter code after removing trailing spaces
      std::vector<Slice> trimmed(n);
      for (int i = 0; i < n; i++) {
        trimmed[i] = RemoveTrailingSpaces(keys[i]);
      }
      return builtin_policy_->CreateFilter(&trimmed[i], n, dst);
    }

    bool KeyMayMatch(const Slice& key, const Slice& filter) const {
      // Use builtin bloom filter code after removing trailing spaces
      return builtin_policy_->KeyMayMatch(RemoveTrailingSpaces(key), filter);
    }
  };
```

Advanced applications may provide a filter policy that does not use a bloom filter but uses some other mechanism for summarizing a set of keys. See <code>rocksdb/filter_policy.h</code> for detail.

## Checksums

<code>rocksdb</code> associates checksums with all data it stores in the file system. There are two separate controls provided over how aggressively these checksums are verified:

<ul>
<li> 
<code>ReadOptions::verify_checksums</code> forces checksum verification of all data that is read from the file system on behalf of a particular read. This is on by default.

<li> <code>Options::paranoid_checks</code> may be set to true before opening a database to make the database implementation raise an error as soon as it detects an internal corruption. Depending on which portion of the database has been corrupted, the error may be raised when the database is opened, or later by another database operation. By default, paranoid checking is off so that the database can be used even if parts of its persistent storage have been corrupted.

If a database is corrupted (perhaps it cannot be opened when paranoid checking is turned on), the <code>rocksdb::RepairDB</code> function may be used to recover as much of the data as possible.

</ul>


## Compaction

You can read more on Compactions here:
<a href="https://github.com/facebook/rocksdb/wiki/Rocksdb-Architecture-Guide#multi-threaded-compactions">Multi-threaded compactions</a>

Here we give overview of the options that impact behavior of Compactions:
<ul>

<li><code>Options::compaction_style</code> - RocksDB currently supports two compaction algorithms - Universal style and Level style. This option switches between the two. Can be kCompactionStyleUniversal or kCompactionStyleLevel. If this is kCompactionStyleUniversal, then you can configure universal style parameters with <code>Options::compaction_options_universal</code>.

<li><code>Options::disable_auto_compactions</code> - Disable automatic compactions. Manual compactions can still be issued on this database.

<li><code>Options::compaction_filter</code> - Allows an application to modify/delete a key-value during background compaction. The client must provide compaction_filter_factory if it requires a new compaction filter to be used for different compaction processes. Client should specify only one of filter or factory.

<li><code>Options::compaction_filter_factory</code> - a factory that provides compaction filter objects which allow an application to modify/delete a key-value during background compaction.
</ul>

Other options impacting performance of compactions and when they get triggered are:
<ul>

<li> <code>Options::access_hint_on_compaction_start</code> - Specify the file access pattern once a compaction is started. It will be applied to all input files of a compaction. Default: NORMAL

<li> <code>Options::level0_file_num_compaction_trigger</code> - Number of files to trigger level-0 compaction. A negative value means that level-0 compaction will not be triggered by number of files at all.

<li> <code>Options::max_mem_compaction_level</code> - Maximum level to which a new compacted memtable is pushed if it does not create overlap. We try to push to level 2 to avoid the relatively expensive level 0=>1 compactions and to avoid some expensive manifest file operations. We do not push all the way to the largest level since that can generate a lot of wasted disk space if the same key space is being repeatedly overwritten.

<li> <code>Options::target_file_size_base</code> and <code>Options::target_file_size_multiplier</code> - Target file size for compaction. target_file_size_base is per-file size for level-1. Target file size for level L can be calculated by target_file_size_base * (target_file_size_multiplier ^ (L-1)) For example, if target_file_size_base is 2MB and target_file_size_multiplier is 10, then each file on level-1 will be 2MB, and each file on level 2 will be 20MB, and each file on level-3 will be 200MB. Default target_file_size_base is 2MB and default target_file_size_multiplier is 1.

<li> <code>Options::expanded_compaction_factor</code> - Maximum number of bytes in all compacted files. We avoid expanding the lower level file set of a compaction if it would make the total compaction cover more than (expanded_compaction_factor * targetFileSizeLevel()) many bytes.

<li> <code>Options::source_compaction_factor</code> - Maximum number of bytes in all source files to be compacted in a single compaction run. We avoid picking too many files in the source level so that we do not exceed the total source bytes for compaction to exceed (source_compaction_factor * targetFileSizeLevel()) many bytes. Default:1, i.e. pick maxfilesize amount of data as the source of a compaction.

<li> <code>Options::max_grandparent_overlap_factor</code> - Control maximum bytes of overlaps in grandparent (i.e., level+2) before we stop building a single file in a level->level+1 compaction.

<li> <code>Options::disable_seek_compaction</code> - Disable compaction triggered by seek. With bloomfilter and fast storage, a miss on one level is very cheap if the file handle is cached in table cache (which is true if max_open_files is large).

<li> <code>Options::max_background_compactions</code> - Maximum number of concurrent background jobs, submitted to the default LOW priority thread pool
</ul>


You can learn more about all of those options in <code>rocksdb/options.h</code>

##  Universal style compaction specific settings

If you're using Universal style compaction, there is an object <code>CompactionOptionsUniversal</code> that hold all the different options for that compaction. The exact definition is in <code>rocksdb/universal_compaction.h</code> and you can set it in <code>Options::compaction_options_universal</code>. Here we give short overview of options in <code>CompactionOptionsUniversal</code>: <ul>

<li> <code>CompactionOptionsUniversal::size_ratio</code> - Percentage flexibility while comparing file size. If the candidate file(s) size is 1% smaller than the next file's size, then include next file into this candidate set. Default: 1

<li> <code>CompactionOptionsUniversal::min_merge_width</code> - The minimum number of files in a single compaction run. Default: 2

<li> <code>CompactionOptionsUniversal::max_merge_width</code> - The maximum number of files in a single compaction run. Default: UINT_MAX

<li> <code>CompactionOptionsUniversal::max_size_amplification_percent</code> - The size amplification is defined as the amount (in percentage) of additional storage needed to store a single byte of data in the database. For example, a size amplification of 2% means that a database that contains 100 bytes of user-data may occupy upto 102 bytes of physical storage. By this definition, a fully compacted database has a size amplification of 0%. Rocksdb uses the following heuristic to calculate size amplification: it assumes that all files excluding the earliest file contribute to the size amplification. Default: 200, which means that a 100 byte database could require upto 300 bytes of storage.

<li> <code>CompactionOptionsUniversal::compression_size_percent</code> - If this option is set to be -1 (the default value), all the output files will follow compression type specified. If this option is not negative, we will try to make sure compressed size is just above this value. In normal cases, at least this percentage of data will be compressed. When we are compacting to a new file, here is the criteria whether it needs to be compressed: assuming here are the list of files sorted by generation time: [ A1...An B1...Bm C1...Ct ], where A1 is the newest and Ct is the oldest, and we are going to compact B1...Bm, we calculate the total size of all the files as total_size, as well as the total size of C1...Ct as total_C, the compaction output file will be compressed iff total_C / total_size < this percentage

<li> <code>CompactionOptionsUniversal::stop_style</code> - The algorithm used to stop picking files into a single compaction run. Can be kCompactionStopStyleSimilarSize (pick files of similar size) or kCompactionStopStyleTotalSize (total size of picked files > next file).

Default: kCompactionStopStyleTotalSize
</ul>

## Thread pools

A thread pool is associated with Env environment object. The client has to create a thread pool by setting the number of background threads using method <code>Env::SetBackgroundThreads()</code> defined in <code>rocksdb/env.h</code>. We use the thread pool for compactions and memtable flushes. Since memtable flushes are in critical code path (stalling memtable flush can stall writes, increasing p99), we suggest having two thread pools - with priorities HIGH and LOW. Memtable flushes can be set up to be scheduled on HIGH thread pool. There are two options available for configuration of background compactions and flushes:
<ul>

<li> <code>Options::max_background_compactions</code> - Maximum number of concurrent background jobs, submitted to the default LOW priority thread pool

<li> <code>Options::max_background_flushes</code> - Maximum number of concurrent background memtable flush jobs, submitted to the HIGH priority thread pool. By default, all background jobs (major compaction and memtable flush) go to the LOW priority pool. If this option is set to a positive number, memtable flush jobs will be submitted to the HIGH priority pool. It is important when the same Env is shared by multiple db instances. Without a separate pool, long running major compaction jobs could potentially block memtable flush jobs of other db instances, leading to unnecessary Put stalls.
</ul>

```cpp
  #include "rocksdb/env.h"
  #include "rocksdb/db.h"

  auto env = rocksdb::Env::Default();
  env->SetBackgroundThreads(2, rocksdb::Env::LOW);
  env->SetBackgroundThreads(1, rocksdb::Env::HIGH);
  rocksdb::DB* db;
  rocksdb::Options options;
  options.env = env;
  options.max_background_compactions = 2;
  options.max_background_flushes = 1;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  assert(status.ok());
  ...
```


## Approximate Sizes

The <code>GetApproximateSizes</code> method can used to get the approximate number of bytes of file system space used by one or more key ranges.

```cpp
   rocksdb::Range ranges[2];
   ranges[0] = rocksdb::Range("a", "c");
   ranges[1] = rocksdb::Range("x", "z");
   uint64_t sizes[2];
   rocksdb::Status s = db->GetApproximateSizes(ranges, 2, sizes);
```

The preceding call will set <code>sizes[0]</code> to the approximate number of bytes of file system space used by the key range <code>[a..c)</code> and <code>sizes[1]</code> to the approximate number of bytes used by the key range <code>[x..z)</code>.


## Environment

All file operations (and other operating system calls) issued by the <code>rocksdb</code> implementation are routed through a <code>rocksdb::Env</code> object. Sophisticated clients may wish to provide their own <code>Env</code> implementation to get better control. For example, an application may introduce artificial delays in the file IO paths to limit the impact of <code>rocksdb</code> on other activities in the system.

```cpp
  class SlowEnv : public rocksdb::Env {
    .. implementation of the Env interface ...
  };

  SlowEnv env;
  rocksdb::Options options;
  options.env = &env;
  Status s = rocksdb::DB::Open(options, ...);
```


## Porting

<code>rocksdb</code> may be ported to a new platform by providing platform specific implementations of the types/methods/functions exported by <code>rocksdb/port/port.h</code>. See <code>rocksdb/port/port_example.h</code> for more details.

In addition, the new platform may need a new default <code>rocksdb::Env</code> implementation. See <code>rocksdb/util/env_posix.h</code> for an example.


## Statistics

To be able to efficiently tune your application, it is always helpful if you have access to usage statistics. You can collect those statistics by setting <code>Options::table_properties_collectors</code> or <code>Options::statistics</code>. For more information, refer to <code>rocksdb/table_properties.h</code> and <code>rocksdb/statistics.h</code>. These should not add significant overhead to your application and we recommend exporting them to other monitoring tools.


## Purging WAL files

By default, old write-ahead logs are deleted automatically when they fall out of scope and application doesn't need them anymore. There are options that enable the user to archive the logs and then delete them lazily, either in TTL fashion or based on size limit.

The options are <code>Options::WAL_ttl_seconds</code> and <code>Options::WAL_size_limit_MB</code>. Here is how they can be used:
<ul>
<li>

If both set to 0, logs will be deleted asap and will never get into the archive.
<li>

If <code>WAL_ttl_seconds</code> is 0 and WAL_size_limit_MB is not 0, WAL files will be checked every 10 min and if total size is greater then <code>WAL_size_limit_MB</code>, they will be deleted starting with the earliest until size_limit is met. All empty files will be deleted.
<li>

If <code>WAL_ttl_seconds</code> is not 0 and WAL_size_limit_MB is 0, then WAL files will be checked every <code>WAL_ttl_seconds / 2</code> and those that are older than WAL_ttl_seconds will be deleted.
<li>

If both are not 0, WAL files will be checked every 10 min and both checks will be performed with ttl being first.
</ul>


## Other Information

Details about the <code>rocksdb</code> implementation may be found in the following documents:
* [RocksDB Overview and Architecture](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics)
* [Format of an immutable Table file](https://github.com/facebook/rocksdb/wiki/Rocksdb-Table-Format)
* <a href="log_format.txt">Format of a log file</a>
</ul>
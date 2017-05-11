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
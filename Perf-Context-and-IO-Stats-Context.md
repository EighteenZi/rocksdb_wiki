Perf Context / IO Stats Context

Perf Context and IO Stats Context can help us understand performance bottleneck of your queries. Compared to options.statistics, which stores accumulated statistics across all the operations, Perf Context and IO Stat Context can help us look inside a query.

This is the header file for perf context: https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_context.h
This is the header file for IO Stats Context: https://github.com/facebook/rocksdb/blob/master/include/rocksdb/iostats_context.h
Level of profiling for the two is controlled by the same function in this header file: https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_level.h

Perf Context and IO Stats Context use the same mechanism. The only difference is Perf Context measures functions of RocksDB, while IO Stats Context measures I/O related calls. They need be enabled in the thread where query to profile is executed. After the profile level is higher than disable, RocksDB will update some counters into a thread-local data structure. After the query, we can read the counters from the thread-local data structure.

## How to Use Them
Here is a typical example of using using Perf Context and IO Stats Context:

``` 
#include “rocksdb/iostat_context.h”
#include “rocksdb/perf_context.h”

rocksdb::SetPerfLevel(rocksdb::PerfLevel::kEnableTimeExceptForMutex);

rocksdb::perf_context.Reset();
rocksdb::iostats_context.Reset();

... // run your query

rocksdb::SetPerfLevel(rocksdb::PerfLevel::kDisable);

... // evaluate or report variables of rocksdb::perf_context and/or rocksdb:iostats_context
```
Note that the same perf level is applied to both of Perf Context and IO Stats Context.

You can also call rocksdb::perf_context.ToString() and rocksdb::iostat_context.ToString() for a human-readable report.

## Coming Soon
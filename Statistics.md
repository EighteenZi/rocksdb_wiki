## Usage
Function `CreateDBStatistics()` will create a statistics object. The object can be passed to one or more DB instances. The counters in the object will contain counters that DB or those DBs.

Here is an example to pass it to one DB:

```
Options options;
options.statistics = rocksdb::CreateDBStatistics();
```
`options.statistics` is a shared pointer, so it is safe to pass the ownership to it. Later the users can access the stats through `options.statistics`.

## Stats Level And Performance Costs
Currently, we implement stats using atomic integers and atomic incremental operations. There are non-negligible costs. There are also costs of timing for duration stats. The costs would vary for different workloads and platforms. We usually observe a 5%-10% costs.

Counter `rocksdb.db.mutex.wait.micros` will issue timing function inside DB mutex. If the timing function is expensive, this can hurt overall write throughput. 



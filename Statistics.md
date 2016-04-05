## Usage
Function `CreateDBStatistics()` will create a statistics object. The object can be passed to one or more DB instances. The counters in the object will contain counters that DB or those DBs.

Here is an example to pass it to one DB:

```
Options options;
options.statistics = rocksdb::CreateDBStatistics();
```
`options.statistics` is a shared pointer, so it is safe to pass the ownership to it. Later the users can access the stats through `options.statistics`.

## Stats Level And Performance Costs
Currently, stats are implemented with atomic integers. We issue atomic incremental operations when updating them. The costs of that are non-negligible. We also have some stats of time duration. They require to call timing functions, which can introduce extra costs. The costs vary on different platforms. We usually observe a 5%-10% costs.

We have two stats levels of statistics, `kExceptTimeForMutex` and `kAll`. The only difference is that with `kExceptTimeForMutex`, counter `rocksdb.db.mutex.wait.micros` is not measured. By measuring the counter, we call the timing function inside DB mutex. If the timing function is slow, it can reduce write throughput significantly.



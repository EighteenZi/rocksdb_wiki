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

## Access The Stats
#### Stats Types
There are two types of stats, ticker and histogram.

Ticker type is represented by 64-bit unsigned integer. The value never decreases or resets. Ticker stats are used to measure counters (e.g. "rocksdb.block.cache.hit"), as well as cumulative bytes (e.g. "rocksdb.bytes.written") or time (e.g. "rocksdb.l0.slowdown.micros").

Histogram type measures distribution of a stat across all operations. Take "rocksdb.db.get.micros" as an example. We measure time spent on each Get() operation and calculate the distribution of them for all of them with the histogram. Most of the histograms are for distribution of duration of a DB operation. There are stats for counts and number of bytes too.

#### Print Human Readable String
We can get a human readable string of all the counters by calling `ToString()`.

### Dump Statistics Periodically in information logs
Statistics are automatically dumped to information logs, for periodic interval of `options.stats_dump_period_sec`. Notice currently it is only dumped after compactions. So if the database doesn't server any write, statistics will not be dumped, despite of `options.stats_dump_period_sec`.

#### Access Stats Programmatically
We can also access specific stat directly from the statistics object. The list of ticker types can be found in enum Tickers. By calling statistics.getTickerCount() for a ticker type, we can retrieve the value. Similarly, single histogram stat can be queried by calling statistics.histogramData() with enum Histograms, or statistics.getHistogramString().

#### 
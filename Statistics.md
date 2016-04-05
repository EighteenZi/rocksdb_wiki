## Usage
Function `CreateDBStatistics()` creates a statistics object. 

Here is an example to pass it to one DB:

```
Options options;
options.statistics = rocksdb::CreateDBStatistics();
```
Technically, you can create a statistics object and pass to multiple DBs. Then the statistics object will contain aggregated values for all those DBs. Note that some stats may loss meaning with this setting, such as "rocksdb.sequence.number".

Advanced users can implement their own statistics class. See the last section for details.

## Stats Level And Performance Costs
Costs of statistics is usually small but non-negligible. We usually observe costs of 5%-10%.

Stats are implemented with atomic integers. We issue atomic incremental operations when updating them. Further more, stats about time duration require to call the timing function. Both of the atomic incremental and the timing function introduce costs, which vary on different platforms. 

We have two stats levels of statistics, `kExceptTimeForMutex` and `kAll`. The only difference is that with `kExceptTimeForMutex`, counter `rocksdb.db.mutex.wait.micros` is not measured. By measuring the counter, we call the timing function inside DB mutex. If the timing function is slow, it can reduce write throughput significantly.

## Access The Stats
#### Stats Types
There are two types of stats, ticker and histogram.

Ticker type is represented by 64-bit unsigned integer. The value never decreases or resets. Ticker stats are used to measure counters (e.g. "rocksdb.block.cache.hit"), cumulative bytes (e.g. "rocksdb.bytes.written") or time (e.g. "rocksdb.l0.slowdown.micros").

Histogram type measures distribution of a stat across all operations. Most of the histograms are for distribution of duration of a DB operation. Taking "rocksdb.db.get.micros" as an example, we measure time spent on each Get() operation and calculate the distribution for all of them.

#### Print Human Readable String
We can get a human readable string of all the counters by calling `ToString()`.

### Dump Statistics Periodically in information logs
Statistics are automatically dumped to information logs, for periodic interval of `options.stats_dump_period_sec`. Note that currently it is only dumped after a compaction. So if the database doesn't serve any write for a long time, statistics may not be dumped, despite of `options.stats_dump_period_sec`.

#### Access Stats Programmatically
We can also access specific stat directly from the statistics object. The list of ticker types can be found in enum Tickers. By calling statistics.getTickerCount() for a ticker type, we can retrieve the value. Similarly, single histogram stat can be queried by calling statistics.histogramData() with enum Histograms, or statistics.getHistogramString().

#### Stats For Time-Interval
All the stats in statistics are cumulative since the opening of the DB. If you need to monitor or report it on time-interval basis, you can check the value periodically and compute the time interval value by taking the difference between the current value and the previous value.

## User-Defined Statistics
Statistics is an abstract class and users can implement their own class and pass it to options.statistics. This is useful when you want to integrate RocksDB's stats to your own stats system. When you implement a user-defined statistics, be aware of the volume of calls to recordTick() and measureTime() by RocksDB. The user-defined stats can easily be the performance bottleneck if not implemented carefully.
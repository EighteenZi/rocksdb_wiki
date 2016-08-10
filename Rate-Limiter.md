When using RocksDB, users maybe want to throttle the maximum write speed within a certain limit for lots of reasons. For example, flash writes cause terrible spikes in read latency if they exceed a certain threshold. Since you've been reading this site, I believe you already know why you need a rate limiter. Actually, 
RocksDB contains a native [RateLimiter](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/rate_limiter.h) which should be adequate for most use cases. 

## How to use
Create a RateLimiter object by calling `NewGenericRateLimiter`, which can be created saperately for each RocksDB instance or by shared among RocksDB instances to control the aggregated write rate of flush and compaction.
```cpp
RateLimiter* rate_limiter = NewGenericRateLimiter(
    rate_bytes_per_sec /* int64_t */, 
    refill_period_us /* int64_t */,
    fairness /* int32_t */);
```
Params:
* `rate_bytes_per_sec`: this is the only parameter you want to set most of the time. It controls the total write rate of compaction and flush in bytes per second. Currently, RocksDB does not enforce rate limit for anything other than flush and compaction, e.g. write to WAL
* `refill_period_us`: this controls how often tokens are refilled. For example, when rate_bytes_per_sec is set to 10MB/s and refill_period_us is set to 100ms, then 1MB is refilled every 100ms internally. Larger value can lead to burst writes while smaller value introduces more CPU overhead. The default value 100,000 should work for most cases.
* `fairness`: RateLimiter accepts high-pri requests and low-pri requests. A low-pri request is usually blocked in favor of hi-pri request. Currently, RocksDB assigns low-pri to request from compaciton and high-pri to request from flush. Low-pri requests can get blocked if flush requests come in continuously. This fairness parameter grants low-pri requests permission by 1/fairness chance even though high-pri requests exist to avoid starvation. You should be good by leaving it at default 10.

Although tokens are refilled with a certain interval set by `refill_period_us`, the maximum bytes can be granted in a single burst has to be bounded since we are not happy to see that tokens are accumulated for a long time and then consumed by a single burst request which definitely does not agree with our intention. `GetSingleBurstBytes()` returns this upper bound of tokens. 

Then each time token should be requested before writes happen. If this request can not be satisfied now, the call will be blocked until tokens get refilled to fulfill the request. For example,
```cpp
// block if tokens are not enough
rate_limiter->Request(1024 /* bytes */, rocksdb::Env::IO_HIGH); 
Status s = db->Flush();
```
Users could also dynamically change rate limiter's bytes per second with `SetBytesPerSecond()` when they need. see [include/rocksdb/rate_limiter.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/rate_limiter.h) for more API details.

## Customization
For the users whose requirements are beyond the functions provided by RocksDB native Ratelimiter, they can implement there own Ratelimiter by extending [include/rocksdb/rate_limiter.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/rate_limiter.h)

RocksDB supports both optimistic and pessimistic concurrency controls. The pessimistic transactions make use of locks to provide isolation between the transactions. The default write policy in pessimistic transactions is _WriteCommitted_, which means that the data is written to the DB, i.e., the memtable, only after the transaction is committed. This policy simplified the implementation but came with some limitations in throughput, transaction size, and variety in supported isolation levels. In the below, we explain these in detail and present the other write policies, _WritePrepared_ and _WriteUnprepared_. We then dive into the design of _WritePrepared_ transactions.

> _WritePrepared_ are to be announced as production-ready soon.

# _WriteCommitted_, Pros and Cons

With _WriteCommitted_ write policy, the data is written to the memtable only after the transaction commits. This greatly simplifies the read path as any data that is read by other transactions can be assumed to be committed. This write policy, however, implies that the writes are buffered in memory in the meanwhile. This makes memory a bottleneck for large transactions. The delay of the commit phase in 2PC (two-phase commit) also becomes noticeable since most of the work, i.e., writing to memtable, is done at the commit phase. When the commit of multiple transactions are done in a serial fashion, such as in 2PC implementation of MySQL, the lengthy commit latency becomes a major contributor to lower throughput. Moreover this write policy cannot provide weaker isolation levels, such as READ UNCOMMITTED, that could potentially provide higher throughput for some applications.

# Alternatives: _WritePrepared_ and _WriteUnprepared_

To tackle the lengthy commit issue, we should do memtable writes at earlier phases of 2PC so that the commit phase become lightweight and fast. 2PC is composed of Write stage, where the transaction `::Put` is invoked, the prepare phase, where `::Prepare` is invoked (upon which the DB promises to commit the transaction if later is requested), and commit phase, where `::Commit` is invoked and the transaction writes become visible to all readers. To make the commit phase lightweight, the memtable write could be done at either `::Prepare` or `::Put` stages, resulting into _WritePrepared_ and _WriteUnprepared_ write policies respectively. The downside is that when another transaction is reading data, it would need a way to tell apart which data is committed, and if they are, whether they are committed before the transaction's start, i.e., in the read snapshot of the transaction. _WritePrepared_ would still have the issue of buffering the data, which makes the memory the bottleneck for large transactions. It however provides a good milestone for transitioning from _WriteCommitted_ to _WriteUnprepared_ write policy. Here we explain the design of _WritePrepared_ policy. We will cover the changes that make the design to also supported _WriteUnprepared_ in an upcoming post.

# _WritePrepared_ in a nutshell

These are the primary design questions that needs to be addressed:
1) How do we identify the key/values in the DB with transactions that wrote them?
2) How do we figure if a key/value written by transaction Txn_w is in the read snapshot of the reading transaction Txn_r?
3) How do we rollback the data written by aborted transactions?

With _WritePrepared_, a transaction still buffers the writes in a write batch object in memory. When 2PC `::Prepare` is called, it writes the in-memory write batch to the WAL (write-ahead log) as well as to the memtable(s) (one memtable per column family); We reuse the existing notion of sequence numbers in RocksDB to tag all the key/values in the same write batch with the same sequence number, `prepare_seq`, which is also used as the identifier for the transaction. At commit time, it writes a commit marker to the WAL, whose sequence number, `commit_seq`, will be used as the commit timestamp of the transaction. Before releasing the commit sequence number to the readers, it stores a mapping from `prepare_seq` to `commit_seq` in an in-memory data structure that we call _CommitCache_. When a transaction reading values from the DB (tagged with `prepare_seq`) it makes use of the _CommitCache_ to figure if `commit_seq` of the value is in its read snapshot. To rollback an aborted transaction, we apply the status before the transaction by making another write that cancels out the writes of the aborted transaction.

The _CommitCache_ is a lock-free data structure that caches the recent commit entries. Looking up the entries in the cache must be enough for almost all th transactions that commit in a timely manner. When evicting the older entries from the cache, it still maintains some other data structures to cover the corner cases for transactions that takes abnormally too long to finish. We will cover them in the design details below.

# _WritePrepared_ Design

Here we present the design details for _WritePrepared_ transactions. We start by presenting the efficient design for _CommitCache_, and dive into other data structures as we see their importance to guarantee the correctness on top of _CommitCache_.

## _CommitCache_

The question of whether a data is committed or not is mainly about very recent transactions. In other words, given a proper rollback algorithm in place, we can assume any old data in the DB is committed and also is present in the snapshot of a reading transaction, which is mostly a recent snapshot. Leveraging this observation, maintaining a cache of recent commit entries must be sufficient for most of the cases. _CommitCache_ is an efficient data structure that we designed for this purpose.

_CommitCache_ is a fixed-size, in-memory array of commit entries. To update the cache with a commit entry, we first index the `prepare_seq` with the array size and then rewrite the corresponding entry in the array, i.e.: _CommitCache_[`prepare_seq` % `array_size`] = <`prepare_seq`, `commit_seq`>. Each insertion will result into eviction of the previous value, which results into updating `max_evicted_seq`, the maximum evicted sequence number from _CommitCache_. When looking up in the _CommitCache_, if a `prepare_seq` > `max_evicted_seq` and yet not in the cache, then it is considered as not committed. If the entry is otherwise found in the cache, then it is committed and will be read by the transaction with snapshot sequence number `snap_seq` if `commit_seq` <= `snap_seq`. If `prepare_seq` < `max_evicted_seq`, then we are reading an old data, which is most likely committed unless proven otherwise, which we explain below how.

Given 100K tps (transactions per second) throughput, 10M entries in the commit cache, and having sequence numbers increased by two per transaction, it would take roughly 50 seconds for an inserted entry to be evicted from the _CommitCache_. In practice however the delay between prepare and commit is a fraction of a millisecond and this limit is thus not likely to be met. For the sake of correctness however we need to cover the cases where a prepared transaction is not committed by the time `max_evicted_seq` advances its `prepare_seq`, as otherwise the reading transactions would assume it is committed. To do so, we maintain a heap of prepare sequence numbers called _PrepareHeap_: a `prepare_seq` is inserted upon `::Prepare` and removed upon `::Commit`. When `max_evicted_seq` advances, if it becomes larger than the minimum `prepare_seq` in the heap, we pop such entries and store them in a set called OldPrepared. Verifying that OldPrepared is empty is an efficient operation which needs to be done before calling an old `prepare_seq` as committed. Otherwise, the reading transactions should also look into OldPrepared to see if the `prepare_seq` of the values that they read is found there. Let us emphasis that such cases is not expected to happen in a reasonable setup and hence not negatively affecting the performance.

Although for read-write transactions, they are expected to commit in fractions of a millisecond after the `::Prepare` phase, it is still possible for a few read-only transactions to hang on some very old snapshots. This is the case for example when a transaction takes a backup of the DB, which could take hours to finish. Such read-only transactions cannot assume an old data to be in their reading snapshot, since their snapshot could also be quite old. More precisely, we still need to keep an evicted entry <`prepare_seq`, `commit_seq`> around if there is a live snapshot with sequence number `snap_seq` where `prepare_seq` <= `snap_seq` < `commit_seq`. In such cases we add the `prepare_seq` to _OldCommit_Map, a mapping from snapshot sequence number to a set of `prepare_seq`. The only transactions that would have to pay the cost of looking into this data structure are the ones that are reading from a very old snapshot. The size of this data structure is expected to be very small as i) there are only a few transactions doing backups and ii) there are limited number of concurrent transactions that overlap with their reading snapshot. _OldCommit_Map is garbage collected upon release of such snapshots.

## _PrepareHeap_

As its name suggests, _PrepareHeap_ is a heap of prepare sequence numbers: a `prepare_seq` is inserted upon `::Prepare` and removed upon `::Commit`. The heap structure allow efficient check of the minimum `prepare_seq` against `max_evicted_seq` as was explained above. To allow efficient removal of entries from the heap, the actual removal is delayed until the entry reaches the top of the heap, i.e., becomes the minimum. To do so, the removed entry will be added to another heap if it is not already on top. Upon each change, the top of the two heaps are compared to see if the top of the main heap is tagged for removal.

## Rollback

To rollback an aborted transaction, for each written key/value we write another key/value to cancel out the previous write. Thanks to write-write conflict avoidance done via locks in pessimistic transactions, it is guaranteed that only one pending write will be on each key, meaning that we only need to look at the previous value to figure the state to which we should rollback. If the result of `::Get` from a snapshot with sequence number `snap_seq`, where `snap_seq` = `prepare_seq` - 1, is a normal value then we do a `::Put` on that key with that value, and if it is a non-existent value, we insert a ::Delete entry. The new values are then committed as another transaction, and the `prepare_seq` of aborted transaction is removed from _PrepareHeap_.

Special care needs to be taken if there is a live snapshot after `prepare_seq` but before the sequence number of the newly written value since they might end up assuming the rolled back value is actually committed, as they will not find it in OldPrepared if `max_evicted_seq` advances the `prepare_seq`. This is currently not an issue with the way we do rollbacks of prepared transactions in MySQL: a prepared transaction could be rolled back only a crash and before new transactions start, which implies that there is no live snapshot at the time of rollback. We will soon extend the implementation of rollback to also cover live snapshots, a feature that will be a must for transaction run under _WriteUnprepared_ write policy.

## Atomic Commit

During a commit, a commit marker is written to the WAL and also a commit entry is added to the _CommitCache_. These two needs to be done atomically otherwise a reading transaction at one point might miss the update into the _CommitCache_ but later sees that. We achieve that by updating the _CommitCache_ before publishing the sequence number of the commit entry. In this way, if a reading snapshot can see the commit sequence number it is guaranteed that the _CommitCache_ is already updated as well. This is done via a `PreReleaseCallback` that is added to `::WriteImpl` logic for this purpose.

## Flush/Compaction

We provide a `IsInSnapshot(prepare_seq, commit_seq)` interface on top of the _CommitCache_ and related data structures, which can be used by the reading transactions. Flush/Compaction threads are also considered a reader and use the same API to figure which versions can be safely garbage collected without affecting the live snapshots.

# Optimizations

Here we cover the additional details in the design that were critical in achieving good performance.

## Lock-free _CommitCache_

Every read from recent data results into a lookup into _CommitCache_. It is therefore vital to make _CommitCache_ efficient for reads. Using a fixed array was already a concious design decision to serve this purpose. We however need to further avoid the overhead of synchronization for reading from this array. To achieve this, we make _CommitCache_ an array of std::atomic<uint64_t> and encode <`prepare_seq`, `commit_seq`> into the available 64 bits. The reads and writes from the array are done with std::memory_order_acquire and std::memory_order_release respectively. In a x86_64 architecture these operations are translated into simple reads and writes into memory thanks to the guarantees of the hardware cache coherency protocol. We have other designs for this data structure and will explore them in future.

To encode <`prepare_seq`, `commit_seq`> into 64 bits we use this algorithm: i) the higher bits of `prepare_seq` is already implied by the index of the entry in the _CommitCache_; ii) the lower bits of prepare seq are encoded in the higher bits of the 64-bit entry; iii) the difference between the `commit_seq` and `prepare_seq` is encoded into the lower bits of the 64-bit entry.

## Less-frequent updates to `max_evicted_seq`

Normally `max_evicted_seq` is expected to be updated upon each eviction from the _CommitCache_. Although updating `max_evicted_seq` is not necessarily expensive, the maintenance operations that comes with it are. For example it requires holding a mutex to verify the top in _PrepareHeap_ (although this can be optimized to be done without a mutex). More importantly it involves holding the db mutex for fetching the list of live snapshots from the DB since maintaining _OldCommit_ depends on the list of live snapshots up to `max_evicted_seq`. To reduce this overhead, upon each update to `max_evicted_seq` we increase its value further by 1% of the _CommitCache_ size, so that the maintenance is done 100 times before _CommitCache_ array wraps around rather than once per eviction.

## Lock-free Snapshot List

In the above, we mentioned that a few read-only transactions doing backups is expected at each point of time. Each evicted entry, which is expected upon each insert, is therefore needs to be checked against the list of live snapshots (which was taken when `max_evicted_seq` was last advanced). Since this is done frequently it needs to be done efficiently, without holding any mutex. We therefore design a data structure that lets us perform this check in a lock-free manner. To do so, we store the first S snapshots in an array of std::atomic<uint64_t>, which we know is efficient for reads and write on x86_64 architecture. The single writer updates the array with a list of snapshots sorted in ascending order by starting from index 0, and updates the size in an atomic variable.

We need to guarantee that the concurrent reader will be able to read all the snapshots that are still valid after the update. Both new and old lists are sorted and the new list is a subset of the previous list plus some new items. Thus if a snapshot repeats in both new and old lists, it will appear with a lower index in the new list. So if we simply insert the new snapshots in order, if an overwritten item is still valid in the new list, it is either written to the same place in the array or it is written in a place with a lower index before it gets overwritten by another item. This guarantees a reader that reads the array from the other side will eventually see a snapshot that repeats in the update, either before it gets overwritten by the writer or afterwards. 

If the number of snapshots exceed the array size, the remaining updates will be stored in a vector protected by a mutex. This is just to ensure correctness in the corner cases and is not expected to happen in a normal run.

# Preliminary Results

The full experimental results are to be reported soon. Here we present the improvement in tps observed in some preliminary experiments with MyRocks:
* sysbench update-noindex: 25%
* sysbench read-write: 7.6%
* linkbench: 3.7%

# Current Limitations

By default RocksDB assigns a separate sequence number to each key/value. This allows for duplicate keys in transactions as they will be simply inserted with separate sequence numbers. With _WritePrepared_ policy, all the key/values in a transaction are inserted with the same sequence number, which would make problems for duplicate keys. We currently have a mechanism to detect the duplicate keys and collapse the write batch to get around this problem. This solution however currently does not cover the cases where the last occurrence of the duplicate key is a Merge. This mechanism is applied only to WriteBatchWithIndex which means that the `::CommitBatch` interface is not covered and the user must ensure there is not duplication when using `::CommitBatch` interface.

Currently we assume that there is no live snapshot when a transactions are rolled back. This is the case in the way MySQL rolls back transactions only after a crash. We will generalize the rollback logic so that it can be performed at any point in the lifetime of the DB.

Currently `Iterator::Refresh` is not supported. There is no fundamental obstacle and it can be added upon request.

Although the DB generated by _WritePrepared_ policy is backward/forward compatible with the classic _WriteCommitted_ policy, the WAL format is not. Therefore to change the _WritePolicy_ the WAL has to be empties first by flushing the DB.
In many cases, people want to drop a range of keys. For example, in MyRocks, we encoded rows from one table by prefixing with table ID. So that when we need to drop a table, all keys with the prefix needs to be dropped. As another example, if a user stores different attributes of a user with key with the format "[user_id][attribute_id]", then if the user deletes the account, we need to delete all the keys prefixing "[user_id]". 

The standard way of deleting those rows is to iterate all the keys and issue Delete() to those keys one by one. This approach works when the number of keys to delete is not large. However, there are two potential concerns of the solution:

1. the space occupied by the data will not be reclaimed immediately. We'll wait for compaction to clean up the data. This is usually a concern when the range to delete takes a significant amount of space of the database.
2. the chunk of tombstones may slow down iterators; 

There are two other ways you can do to delete keys from a range:

You can issue a DeleteFilesInRange() to the range. The command will remove all SST files only containing keys in the range to delete. For a large chunk, it will immediately reclaim most space. It is a good solution to problem 1. After the operations, some keys in the range may still exist in the database. You make follow up operations to further remove them in a slower pace. Also be aware that, DeleteFilesInRange() will be removed despite of existing snapshots. So you shouldn't expect to be able to read data from the range using existing snapshots any more.

Another solution is to apply compaction filter together with CompactRange(). You can write a compaction filter that can filter out keys from deleted range. When you want to delete keys from a range, call CompactRange() for the range to delete. While the compaction finishes, the keys will be dropped. We recommend you to turn CompactionFilter::IgnoreSnapshots() to true to make sure keys are dropped even if you have outstanding snapshots. Otherwise, you may not be able to fully remove all the keys in the range from the system. This approach can also solve the problem of reclaiming data, but it issues extra I/Os than the DeleteFilesInRange(). However, DeleteFilesInRange() cannot remove all the data in the range. So a better way is to first apply DeleteFilesInRange(), and then issue CompactRange() with compaction filter.

Problem 2 is a harder problem to solve. One way is to apply DeleteFileInRnage() and CompactRange() so that all keys and tombstones for the range are dropped. It works for large ranges, but it is too expensive for frequent dropping of small ranges. If we cannot remove all the data immediately, there are still two ways to reduce the harm:

1. if you never overwrite existing keys, you can try to use DB::SingleDelete() instead of Delete() to kill tombstones after it meets the original keys; 
2. use NewCompactOnDeletionCollectorFactory() to speed up compaction when there are chunks of tombstones.

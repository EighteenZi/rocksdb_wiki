## Introduction
Similar to `Delete()` method, `SingleDelete()` removes the database entry for a _key_, but has the prerequisites that the _key_ exists and was not overwritten. Returns OK on success, and a non-OK status on error.  It is not an error if _key_ did not exist in the database. If a _key_ is overwritten (by calling `Put()` multiple times), then the result of calling `SingleDelete()` on this key is undefined.  `SingleDelete()` only behaves correctly if there has been only one `Put()` for this _key_ since the previous call to `SingleDelete()` for this _key_. This feature is currently an experimental performance optimization for a very specific workload. The following code shows how to use `SingleDelete`:
```cpp
  std::string value;
  rocksdb::Status s;
  db->Put(rocksdb::WriteOptions(), "foo", "bar1");
  db->SingleDelete(rocksdb::WriteOptions(), "foo");
  s = db->Get(rocksdb::ReadOptions(), "foo", &value); // s.IsNotFound()==true
  db->Put(rocksdb::WriteOptions(), "foo", "bar2");
  db->Put(rocksdb::WriteOptions(), "foo", "bar3");
  db->SingleDelete(rocksdb::ReadOptions(), "foo", &value); // Undefined result
```

`SingleDelete` API is also available in `WriteBatch`. Actually, `DB::SingleDelete()` is implemented by creating a `WriteBatch` with only one operation, `SingleDelete`, in this batch. The following code snippet shows the basic usage of `WriteBatch::SingleDelete()`:
```cpp
  rocksdb::WriteBatch batch;
  batch.Put(key1, value);
  batch.SingleDelete(key1);
  s = db->Write(rocksdb::WriteOptions(), &batch);
```

## Notes
* Callers have to ensure that `SingleDelete` only applies to a _key_ having not been deleted using `Delete()` or written using `Merge()`.  Mixing `SingleDelete()` operations with `Delete()` and `Merge()` can result in undefined behavior.
* `SingleDelete` is not supported by HashCuckooRep, which means you should not call `SingleDelete` if you set
`options.memtable_factory` with [`NewHashCuckooRepFactory`](https://github.com/facebook/rocksdb/blob/522de4f59e6314698286cf29d8a325a284d81778/include/rocksdb/memtablerep.h#L325).
* Consider setting `write_options.sync = true`([Asynchronous Writes](https://github.wom/facebook/rocksdb/wiki/Basic-Operations#asynchronous-writes)).

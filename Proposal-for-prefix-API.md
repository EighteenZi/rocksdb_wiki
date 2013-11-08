NOTE: This is just a proposal, not functional yet.

There are a lot of applications that don't care about total ordering of keys in the database. They only care about total ordering of keys with the same prefix. For example, if you keep actions by the user stored in a db, your keys could be something like "user_id:timestamp". Your query would look like "give me all actions of that user", or "give me all the keys with prefix user_id:".

If you specify options.prefix_extractor when opening a DB, RocksDB will optimize data accesses for those kind of applications. Prefix extractor is a way of explaining to RocksDB the structure of your key. It's a function that receives a key and returns a prefix of the key. For more information, refer to rocksdb/options.h

If a option.prefix_extractor is specified, RocksDB will automatically use hashtable memtable representation that optimizes prefix lookups for data in memory (hash table instead of binary search). It will also build bloom filters on top of sst files and avoid I/Os if it's certain that supplied prefix can not be found in that specific sst table.

When you call DB::NewIterator() it will by default return an iterator that guarantees only partial ordering. Doing Seek(prefix) on returned iterator guarantees that you will iterate over all the keys in the DB with specified prefix. It might also return some more keys, but feel free to ignore those. Correct iteration would be:

  auto iter = DB::NewIterator();
  for (iter.Seek(prefix); iter.Valid() && iter.key().startswith(prefix); iter.Next())

You can reuse the iterator for seeking to other prefixes, which we strongly encourage since creating iterators is very costly. However, keep in mind that iterators reflect the state of the database when they were created. You will not see new updates when you Seek() to a new prefix.
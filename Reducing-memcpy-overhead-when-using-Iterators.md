## Use case
In certain scenarios the user may need to Iterate over range of keys and keep them in memory to process them.
A simple example could be something like this

	ReadOptions ro;
	Iterator* iter = db_->NewIterator(ro);

	// Get the keys from the DB
	std::vector<std::string> db_keys;
	for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
		db_keys.push_back(iter->key().ToString());
	}

	// Process the keys (in this case we simply sort them)
	auto key_comparator = [](const std::string& k1, const std::string& k2) {
		if (k1.size() == k2.size()) {
			return k1 < k2;
		}
		return k1.size() < k2.size();
	};
	std::sort(db_keys.begin(), db_keys.end(), key_comparator);

	for (size_t i = 0; i < db_keys.size(); i++) {
		// Use processed keys
	}

	delete iter;

In this example we simply load keys from the DB into memory, sort them using a comparator that is different from DB comparator and then use the sorted keys.

## The Problem
The issue with this approach is in this line

	db_keys.push_back(iter->key().ToString());

If our keys are huge the cost of copying the key from RocksDB into our std::vector will be significant and we cannot escape this overhead since iter->key() Slice will be invalid the moment our Iterator move forward when iter->Next() is called.

## The Solution
We have introduced a new option for Iterators, ReadOptions::pin_data. When setting this option to true, RocksDB Iterator will pin the data blocks and guarantee that the Slice returned by Iterator::key() will be valid as long as the Iterator is not deleted. 

	ReadOptions ro;
	// Tell RocksDB to keep the key Slice valid as long as
	// the Iterator is not deleted
	ro.pin_data = true;
	Iterator* iter = db_->NewIterator(ro);

	// Get the keys from the DB
	std::vector<Slice> db_keys;
	for (iter->SeekToFirst(); iter->Valid(); iter->Next()) {
		// We check "rocksdb.iterator.is-key-pinned" property to make sure that
		// the key is actually pinned
		std::string is_key_pinned;
		iter->GetProperty("rocksdb.iterator.is-key-pinned", &is_key_pinned);
		assert(is_key_pinned == "1");

		// iter->key() Slice will be valid as long as
		// the Iterator is not deleted
		db_keys.push_back(iter->key());
	}

	// Process the keys (in this case we simply sort them)
	auto key_comparator = [](const Slice& k1, const Slice& k2) {
		if (k1.size() == k2.size()) {
			return k1.compare(k2) < 0;
		}
		return k1.size() < k2.size();
	};

	std::sort(db_keys.begin(), db_keys.end(), key_comparator);

	for (size_t i = 0; i < db_keys.size(); i++) {
		// Use processed keys
	}

	delete iter;

After setting ReadOptions::pin_data to true, now we can use Iterator::key() Slice without copying it

	db_keys.push_back(iter->key());

## Requirements
Right now to use this feature, RocksDB must be created using BlockBased table with BlockBasedTableOptions::use_delta_encoding set to false.

	Options options;
	BlockBasedTableOptions table_options;
	table_options.use_delta_encoding = false;
	options.table_factory.reset(NewBlockBasedTableFactory(table_options));

To verify that the current key Slice is pinned and will be valid as long as the Iterator is not deleted,
We can check `rocksdb.iterator.is-key-pinned` Iterator property and assert that it's equal to `1`

	std::string is_key_pinned;
	iter->GetProperty("rocksdb.iterator.is-key-pinned", &is_key_pinned);
	assert(is_key_pinned == "1");
# Prefix Seek API
When options.prefix_extractor for your DB or column family is specified, RocksDB is in a "prefix seek" mode, explained below. Example of how to use it:

```cpp
Options options;

// <---- Enable some features supporting prefix extraction
options.prefix_extractor.reset(NewFixedPrefixTransform(3));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"
iter->Next(); // Find next key-value pair inside prefix "foo"
```

options.prefix_extractor is a shared_pointer of a SliceTransform instance. By calling SliceTransform.Transform(), we extract a Slice representing a substring of the Slice, usually the prefix part. In this wiki page, we use "prefix" to refer to the output of options.prefix_extractor.Transform() of a key. You can use fixed length prefix transformer, by calling NewFixedPrefixTransform(prefix_len), or you can implement your own prefix transformer in the way you want and pass it to options.prefix_extractor.

When options.prefix_extractor is not nullptr, iterators are not guaranteed a total order of all keys, but only keys for the same prefix. Furthermore, we only support the forward iterating direction. When doing Iterator.Seek(lookup_key), RocksDB will extract the prefix of lookup_key. If there is one or more keys in the database matching prefix of lookup_key, RocksDB will place the iterator to the key equal or larger than lookup_key of the same prefix, as for total ordering mode. If no key of the prefix equals or is larger than lookup_key, or after calling one or more Next(), we finish all keys for the prefix, we might return Valid()=false, or any key in the DB that is larger than the previous key, depending on whether ReadOptions.prefix_same_as_start=true. From release 5.0, we support Prev() in prefix mode, but only when the iterator is still within the range of all the keys for the prefix. The output of Prev() is not guaranteed to be  correct when the iterator is out of the range.

When prefix seek mode is enabled, RocksDB will freely organize the data or build look-up data structures that can locate keys for specific prefix or rule out non-existing prefixes quickly. Here are some supported optimizations for prefix seek mode: prefix bloom for block based tables and mem tables, [hash-based mem tables](https://github.com/facebook/rocksdb/wiki/Hash-based-memtable-implementations), as well as [PlainTable](https://github.com/facebook/rocksdb/wiki/PlainTable-Format) format. One example setting:

```cpp
Options options;

// Enable prefix bloom for mem tables
options.prefix_extractor.reset(NewFixedPrefixTransform(3));
options.memtable_prefix_bloom_bits = 100000000;
options.memtable_prefix_bloom_probes = 6;

// Enable prefix bloom for SST files
BlockBasedTableOptions table_options;
table_options.filter_policy.reset(NewBloomFilterPolicy(10, true));
options.table_factory.reset(NewBlockBasedTableFactory(table_options));

DB* db;
Status s = DB::Open(options, "/tmp/rocksdb",  &db);

......

auto iter = db->NewIterator(ReadOptions());
iter->Seek("foobar"); // Seek inside prefix "foo"

```

From release 3.5, we support a read option to allow RocksDB to use total order even if options.prefix_extractor is given. To enable the feature set ReadOption.total_order_mode=true to the read option passed when doing NewIterator(), example:

```cpp
ReadOptions read_options;
read_options.total_order_mode = true;
auto iter = db->NewIterator(read_options);
Slice key = "foobar";
iter->Seek(key);  // Seek "foobar" in total order
```

Performance might be worse in this mode. Please aware not all the implementation of prefix seek supports this option. For example, some implementation of [PlainTable](https://github.com/facebook/rocksdb/wiki/PlainTable-Format) doesn't support it and you'll see an error in status code when you try to use it. [Hash-based mem tables](https://github.com/facebook/rocksdb/wiki/Hash-based-memtable-implementations) might do a very expensive online sorting if you use it. This mode is supported in prefix bloom and hash index of block based tables.

# Limitation
SeekToLast() is not supported well with prefix iterating. SeekToFirst() is only supported by some configuration. You should use total order mode, if you will execute those types of queries against your iterator. 

One common bug of using prefix iterating is to use prefix mode to iterate in reverse order. But it is not yet supported. If reverse iterating is your common query pattern, you can reorder the data to turn your iterating order to be forward. You can do it through implementing a customized comparator, or encode your key in a different way.

# API change from 2.8 -> 3.0
In this section, we explained the API as of 2.8 release and the change in 3.0.

## Before the Change

As of RocksDB 2.8, there are 3 seek modes:

### Total Order Seek
This is the traditional seek behavior you'd expect. The seek performs on a total ordered key space, positioning the iterator to a key that is greater or equal to the target key you seek.

```cpp
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);
```

Not all table format supports total order seek. For example, the newly introduced [PlainTable](https://github.com/facebook/rocksdb/wiki/PlainTable-Format) format only supports prefix-based seek() unless it is opened in total order mode (Options.prefix_extractor == nullptr).

### Use ReadOptions.prefix
This is the least flexible way to do a seek. Prefix needs to be supplied when creating an iterator. 

```cpp
Slice prefix = "foo";
ReadOptions ro;
ro.prefix = &prefix;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar"
iter->Seek(key);
```

Options.prefix_extractor is a prerequisite. The `Seek()` is constrained to the prefix provided by `ReadOptions`, which means you will need to create a new iterator to seek a different prefix. The benefit of this approach is that irrelevant files are filtered out at the time of building the new iterator. So if you want to seek multiple keys with the same prefix, it might perform better. However, we consider this is a very rare use case.

### Use ReadOptions.prefix_seek
This mode is more flexible than `ReadOption.prefix`. No pre-filtering is done at iterator creation time. As a result, the same iterator can be reused for seek of different key/prefix.

```cpp
ReadOptions ro;
ro.prefix_seek = true;
auto iter = db->NewIterator(ro);
Slice key = "foo_bar";
iter->Seek(key);
```

Same as ReadOptions.prefix, Options.prefix_extractor is a prerequisite.

## What's Changed
It becomes obvious that 3 modes of seek are confusing:
* One mode would require another option to be set (e.g. `Options.prefix_extractor`);
* It is not obvious to our users which mode of the last two is preferred under different circumstances

This change tries to address this issue and makes things straight: by default, `Seek()` is performed in prefix mode if `Options.prefix_extractor` is defined and vice versa. The motivation is simple: if `Options.prefix_extractor` is provided, it is a very clear signal that underlying data can be sharded and prefix seek is a natural fit. Usage becomes unified:

```cpp
auto iter = db->NewIterator(ReadOptions());
Slice key = "foo_bar";
iter->Seek(key);
```

## Transition to the New Usage
Transition to the new style should be simple: remove the assignment to `Options.prefix` or `Options.prefix_seek`, since they are deprecated. Now, seek directly with your target key or prefix. Since
`Next()` can go across the boundary to a different prefix, you will need to check the end condition:

```cpp
    auto iter = DB::NewIterator(ReadOptions());
    for (iter.Seek(prefix); iter.Valid() && iter.key().starts_with(prefix); iter.Next()) {
       // do something
    }
```
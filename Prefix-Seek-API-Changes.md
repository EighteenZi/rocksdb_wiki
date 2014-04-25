# Before the Change

As of RocksDB 2.8, there are 3 seek modes:

### Total Order Seek
This is the traditional seek behavior you'd expect. The seek performs on a total ordered key space, positioning the iterator to a key that is greater or equal to the target key you seek.

    auto iter = db->NewIterator(ReadOptions());
    Slice key = "foo_bar";
    iter->Seek(key);

Not all table format supports total order seek. For example, the newly introduced [PlainTable](https://github.com/facebook/rocksdb/wiki/PlainTable-Format) format only supports prefix-based seek() unless it is opened in total order mode.

### Use ReadOptions.prefix
This is the least flexible way to do a seek. Prefix needs to be supplied when creating an iterator. 

    Slice prefix = "foo";
    ReadOptions ro;
    ro.prefix = &prefix;
    auto iter = db->NewIterator(ro);
    Slice key = "foo_bar"
    iter->Seek(key);

Options.prefix_extractor is a prerequisite. The `Seek()` is constrained to the prefix provided by `ReadOptions`, which means you will need to create a new iterator to seek a different prefix. The benefit of this approach is that irrelevant files are filtered out at the time of building the new iterator. So if you want to seek multiple keys with the same prefix, it might perform better. However, we consider this is a very rare use case.

### Use ReadOptions.prefix_seek
This mode is more flexible than `ReadOption.prefix`. No pre-filtering is done at iterator creation time. As a result, the same iterator can be reused for seek of different key/prefix.

    ReadOptions ro;
    ro.prefix_seek = true;
    auto iter = db->NewIterator(ro);
    Slice key = "foo_bar";
    iter->Seek(key);

Same as ReadOptions.prefix, Options.prefix_extractor is a prerequisite.

# What's Changed
It becomes obvious that 3 modes of seek are confusing:
* One mode would require another option to be set (e.g. `Options.prefix_extractor`);
* It is not obvious to our users which mode of the last two is preferred under different circumstances

This change tries to address this issue and makes things straight: by default, `Seek()` is performed in prefix mode if `Options.prefix_extractor` is defined and vice versa. The motivation is simple: if `Options.prefix_extractor` is provided, it is a very clear signal that underlying data can be sharded and prefix seek is a natural fit. Usage becomes unified:

    auto iter = db->NewIterator(ReadOptions());
    Slice key = "foo_bar";
    iter->Seek(key);

# Transition to the New Usage
Transition to the new style should be simple: remove the assignment to `Options.prefix` or `Options.prefix_seek`, since they are deprecated. Now, seek directly with your target key or prefix. Since
`Next()` can go across the boundary to a different prefix, you will need to check the end condition:

    auto iter = DB::NewIterator(ReadOptions());
    for (iter.Seek(prefix); iter.Valid() && iter.key().startswith(prefix); iter.Next()) {
       // do something
    }

Users pass options to RocksDB through _Options_ class. Other than setting options in the _Options_ class, there are two other ways to set it:

1. get an option class from an [[option file|RocksDB-Options-File]].
2. get it from an option string by calling
3. get it from a string map

To get an option from a string, call `GetColumnFamilyOptionsFromString()` or `GetDBOptionsFromString()` with a string containing the information. There is also a special `GetBlockBasedTableOptionsFromString()` and `GetPlainTableOptionsFromString()` to get table specific option. 

An example of an option string will like this:
```
write_buffer_size=10;max_write_buffer_number=16;plain_table_factory={user_key_len=66;bloom_bits_per_key=20;};arena_block_size=1024
```
Each option will be given as `<option_name>:<option_value>` separated by `;`. The `<option_name>` always map the option name in the DBOptions and ColumnFamilyOptions class. You can find the list of options and their descriptions in those two classes in the source file [[Options.h|https://github.com/facebook/rocksdb/blob/master/include/rocksdb/options.h]] of the source code of your release. Note that although most of the options are supported in the option string, there are exceptions. You can find the list of supported options in variable `db_options_type_info`, `cf_options_type_info` and `block_based_table_type_info` in the source file [[util/options_helper.h|https://github.com/facebook/rocksdb/blob/master/util/options_helper.h]] of the source code of your release.
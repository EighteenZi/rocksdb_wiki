RocksDB CompactionFilter offers a way to remove / filter expired key / value pairs based on custom logic in background.  Now we have implemented an extension on top of it which allows users to implement their custom CompactionFilter in Lua!  This feature is available in RocksDB 5.0.

# Benefits

Developing CompactionFilter in Lua has the following benefits:

* On-the-fly change: updating Lua Compaction Filter can be done on the fly without shutting down your RocksDB instance.
* Individual binary unit:  as updating Lua Compaction Filter can be done on the fly, it no longer requires rebuilding the binary as required by the C++ compaction filter.  This is a huge benefit for services built on top of RocksDB who maintains custom CompactionFilters for their customers.
* Safer: errors in CompactionFilter will no longer result in core dump.  Lua engine captures all the exceptions.

# How to Use? 

Using RocksLuaCompactionFilter is simple.  All you need to do is the following steps:

* Build RocksDB with LUA_PATH set to the root directory of Lua.
* Config RocksLuaCompactionFilterOptions with your Lua script. (more details are described in the next section)
* Use your RocksLuaCompactionFilterOptions in step 1 to construct a RocksLuaCompactionFilterFactory.
* Pass your RocksLuaCompactionFilterFactory in step 2 to ColumnFamilyOptions::compaction_filter_factory. 

## Example 

Here's a simple RocksLuaCompactionFilter that filter out any keys whose initial is less than `r`:

```cpp
  lua::RocksLuaCompactionFilterOptions lua_opt;
  // removes all keys whose initial is less than 'r'
  lua_opt.lua_script =
      "function Filter(level, key, existing_value)\n"
      "  if key:sub(1,1) < 'r' then\n"
      "    return true, false, \"\"\n"
      "  end\n"
      "  return false, false, \"\"\n"
      "end\n"
      "\n"
      "function FilterMergeOperand(level, key, operand)\n"
      "  return false\n"
      "end\n"
      "function Name()\n"
      "  return \"KeepsAll\"\n"
      "end\n";

  // specify error log.
  auto* rocks_logger = new facebook::rocks::RocksLogger(
      "RocksLuaTest",
      true,  // print error message in GLOG
      true,  // print error message to scribe, available in LogView `RocksDB ERROR`
      nullptr);

  // Create RocksLuaCompactionFilter with the above lua_opt and pass it to Options
  rocksdb::Options options;
  options.compaction_filter_factory =
      std::make_shared<rocksdb::lua::RocksLuaCompactionFilterFactory>(lua_opt);
  ...
  // open FbRocksDB with the above options
  rocksdb::DB* db;
  auto status = openRocksDB(options, "RocksDBWithLua", &db);
```

## Config RocksLuaCompactionFilterOptions 

Here we introduce how to config RocksLuaCompactionFilterOptions in more detail.  The definition of RocksLuaCompactionFilterOptions can be found in include/rocksdb/utilities/lua/rocks_lua_compaction_filter.h.

### Config the Lua Script (RocksLuaCompactionFilter::script) 
The first and the most important parameter is RocksLuaCompactionFilterOptions::script, which is where your Lua compaction filter will be implemented.  Your Lua script must implement the required functions, which are Name() and Filter().

```cpp
  // The lua script in string that implements all necessary CompactionFilter
  // virtual functions.  The specified lua_script must implement the following
  // functions, which are Name and Filter, as described below.
  //
  // 0. The Name function simply returns a string representing the name of
  //    the lua script.  If there's any erorr in the Name function, an
  //    empty string will be used.
  //    --- Example
  //      function Name()
  //        return "DefaultLuaCompactionFilter"
  //      end
  //
  //
  // 1. The script must contains a function called Filter, which implements
  //    CompactionFilter::Filter() , takes three input arguments, and returns
  //    three values as the following API:
  //
  //   function Filter(level, key, existing_value)
  //     ...
  //     return is_filtered, is_changed, new_value
  //   end
  //
  //   Note that if ignore_value is set to true, then Filter should implement
  //   the following API:
  //
  //   function Filter(level, key)
  //     ...
  //     return is_filtered
  //   end
  //
  //   If there're any error in the Filter() function, then it will keep
  //   the input key / value pair.
  //
  //   -- Input
  //   The function must take three arguments (integer, string, string),
  //   which map to "level", "key", and "existing_value" passed from
  //   RocksDB.
  //
  //   -- Output
  //   The function must return three values (boolean, boolean, string).
  //     - is_filtered: if the first return value is true, then it indicates
  //       the input key / value pair should be filtered.
  //     - is_changed: if the second return value is true, then it indicates
  //       the existing_value needs to be changed, and the resulting value
  //       is stored in the third return value.
  //     - new_value: if the second return value is true, then this third
  //       return value stores the new value of the input key / value pair.
  //
  //   -- Examples
  //     -- a filter that keeps all key-value pairs
  //     function Filter(level, key, existing_value)
  //       return false, false, ""
  //     end
  //
  //     -- a filter that keeps all keys and change their values to "Rocks"
  //     function Filter(level, key, existing_value)
  //       return false, true, "Rocks"
  //     end
    
  std::string lua_script;
```

An optimization without using value (RocksLuaCompactionFilter::ignore_value) 
In case your CompactionFilter never uses value to determine whether to keep or discard a key / value pair, then setting RocksLuaCompactionFilterOptions::ignore_value=true and implement the simplified Filter() API.   Our result shows that this optimization can save up to 40% CPU overhead introduced by LuaCompactionFilter:

```cpp
  // If set to true, then existing_value will not be passed to the Filter
  // function, and the Filter function only needs to return a single boolean
  // flag indicating whether to filter out this key or not.
  //
  //   function Filter(level, key)
  //     ...
  //     return is_filtered
  //   end
  bool ignore_value = false;
```

The simplified Filter() API only takes two input arguments and returns only one boolean flag indicating whether to keep or discard the input key.

### Error Log Configuration (RocksLuaCompactionFilterOptions::error_log) 
When RocksLuaCompactionFilter hit any error, it will act as no-op and always return false for any key / value pair that cause errors.  Developers can config RocksLuaCompactionFilterOptions::error_log to log any Lua errors:

```cpp
 // When specified a non-null pointer, the first "error_limit_per_filter"
 // errors of each CompactionFilter that is lua related will be included
 // in this log.
 std::shared_ptr<Logger> error_log;
```

Note that for each compaction job, we only log the first few Lua errors in error_log to avoid generating too many error messages, and the number of errors it reports per compaction job can be configured via error_limit_per_filter.  The default number is one.

```cpp
 // The number of errors per CompactionFilter will be printed
 // to error_log.
 int error_limit_per_filter = 1;
```

## Dynamic Updating Lua Script
To update the Lua script while the RocksDB database is running, simply call the `SetScript()` API of your RocksLuaCompactionFilterFactory:

```cpp
  // Change the Lua script so that the next compaction after this
  // function call will use the new Lua script.
  void SetScript(const std::string& new_script);
``` 
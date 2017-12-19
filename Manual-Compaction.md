Compaction can be triggered manually by calling the <code>DB::CompactRange</code> or <code>DB::CompactFiles</code> method. This is meant to be used by advanced users to implement custom compaction strategies, including but not limited to the following use cases -

1. Optimize for read heavy workloads by compacting to lowest level after ingesting a large amount of data
2. Force the data to go through the compaction filter in order to consolidate it
3. Migrate to a new compaction configuration. For example, if changing number of levels, <code>CompactRange</code> can be called to compact to bottommost level and then move the files to a target level

The example code below shows how to use the APIs.

```cpp
Options dbOptions;

DB* db;
Status s = DB::Open(dbOptions, "/tmp/rocksdb",  &db);

// Write some data
...
Slice begin("key1");
Slice end("key100");
CompactRangeOptions options;

s = db->CompactRange(options, &begin, &end);
```
Or
```
CompactionOptions options;
std::vector<std::string> input_file_names;
int output_level;
...
Status s = db->CompactFiles(options, input_file_names, output_level);
```

## CompactRange

The <code>begin</code> and <code>end</code> arguments define the key range to be compacted. The behavior varies depending on the compaction style being used by the db. In case of universal and FIFO compaction styles, the <code>begin</code> and <code>end</code> arguments are ignored and all files are compacted. Also, files in each level are compacted and left in the same level. For leveled compaction style, all files containing keys in the given range are compacted to the last level containing files. If either <code>begin</code> or <code>end</code> are NULL, it is taken to mean the key before all keys in the db or the key after all keys respectively.

If more than one thread calls manual compaction, only one will actually schedule it while the other threads will simply wait for the scheduled manual compaction to complete. If <code>CompactRangeOptions::exclusive_manual_compaction</code> is set to true, the call will disable scheduling of automatic compaction jobs and wait for existing automatic compaction jobs to finish.
 
The <code>CompactRangeOptions</code> supports the following options -
<ul>

<li><code>CompactRangeOptions::exclusive_manual_compaction</code> When set to true, no other compaction will run when this manual compaction is running. Default value is <code>true</code>

<li><code>CompactRangeOptions::change_level</code>,<code>CompactRangeOptions::target_level</code> Together, these options control the level where the compacted files will be placed. If <code>target_level</code> is -1, the compacted files will be moved to the minimum level whose computed max_bytes is still large enough to hold the files. Intermediate levels must be empty. For example, if the files were initially compacted to L5 and L2 is the minimum level large enough to hold the files, they will be placed in L2 if L3 and L4 are empty or in L4 if L3 is non-empty. If <code>target_level</code> is positive, the compacted files will be placed in that level provided intermediate levels are empty. If any any of the intermediate levels are not empty, the compacted files will be left where they are.

<li><code>CompactRangeOptions::target_path_id</code> Compaction outputs will be placed in options.db_paths[target_path_id] directory.

<li><code>CompactRangeOptions::bottommost_level_compaction</code> When set to <code>BottommostLevelCompaction::kSkip</code>, or when set to <code>BottommostLevelCompaction::kIfHaveCompactionFilter</code> and a compaction filter is defined for the column family, the bottommost level files are not compacted.
</ul>

##  CompactFiles

This API compacts all the input files into a set of output files in the <code>output_level</code>. The number of output files is determined by the size of the data and the setting of <code>CompactionOptions::output_file_size_limit</code>.



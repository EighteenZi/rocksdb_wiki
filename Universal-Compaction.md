## Overview
Universal Compaction Style is a compaction style, targeting the use cases requiring lower write amplification, trading off read amplification and space amplification.

When using this compaction style, all the SST files are organized as sorted runs covering the whole key ranges. One sorted run covers data generated during a time range. Different sorted runs never overlap on their time ranges. Compaction can only happen among two or more sorted runs of adjacent time ranges. The output is a single sorted run whose time range is the combination of input sorted runs. After any compaction, the condition that sorted runs never overlap on their time ranges still holds. A sorted run can be implemented as an L0 file, or a "level" in which data is stored as key range partitioned files.

## Limitations

### Double Size Issue
In universal style compaction, sometimes full compaction is needed. In this case, output data size is similar to input size. During compaction, both of input files and the output file need to be kept, so the DB will be temporarily double the disk space usage. Be sure to keep enough free space for full compaction.

### DB (Column Family) Size if num_levels=1
When using Universal Compaction, if num_levels = 1, all data of the DB (or Column Family to be precise) is sometimes compacted to one single SST file. There is a limitation of size of one single SST file. In RocksDB, a block cannot exceed 4GB (to allow size to be uint32). The index block can exceed the limit if the single SST file is too big. The size of index block depends on your data. In one of our use cases, we would observe the DB to reach the limitation when the DB grows to about 250GB, using 4K data block size.

This problem is mitigated if users set num_levels to be much larger than 1. In that case, larger "files" will be put in larger "levels" with files divided into smaller files (more explanation below). L0->L0 compaction can still happen for parallel compactions, but most likely files in L0 are much smaller.

## Data Layout and Placement
### Sorted Runs
As mentioned above, data is organized as sorted runs. A sorted runs are laid out by updated time of the data in it and stored as either files in L0 or a whole "level".

Here is an example of a typical file layout:
```
Level 0: File0_0, File0_1, File0_2
Level 1: (empty)
Level 2: (empty)
Level 3: (empty)
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```
Levels with a larger number contain older sorted run than levels of smaller numbers. In this example, there are 5 sorted runs: three files in level 0, level 4 and 5. Level 5 is the oldest sorted run, level 4 is newer, and the level 0 files are the newest.

### Placement of Compaction Outputs
Compaction is always scheduled for sorted runs with consecutive time ranges and the outputs are always another sorted run. We always place compaction outputs to the highest possible level, following the rule of older data on levels with larger numbers.

Use the same example shown above. We have following sorted runs:
```
File0_0
File0_1
File0_2
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```
If we compact all the data, the output sorted run will be placed in level 5. so it becomes:
```
Level 5: File5_0', File5_1', File5_2', File5_3', File5_4', File5_5', File5_6', File5_7'
```
Starting from this state, let's see how to place output sorted runs if we schedule different compactions:

If we compact File0_1, File0_2 and Level 4, the output sorted run will be placed in level 4.
```
Level 0: File0_0
Level 1: (empty)
Level 2: (empty)
Level 3: (empty)
Level 4: File4_0', File4_1', File4_2', File4_3'
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

If we compact File0_0, File0_1 and File0_2, the output sorted run will be placed in level 3. 
```
Level 0: (empty)
Level 1: (empty)
Level 2: (empty)
Level 3: File3_0, File3_1, File3_2
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

If we compact File0_0 and File0_1, the output sorted run will still be placed in level 0.
```
Level 0: File0_0', File0_2
Level 1: (empty)
Level 2: (empty)
Level 3: (empty)
Level 4: File4_0, File4_1, File4_2, File4_3
Level 5: File5_0, File5_1, File5_2, File5_3, File5_4, File5_5, File5_6, File5_7
```

### Special case options.num_levels=1
If options.num_levels=1, we still follow the same placement rule. It means all the files will be placed under level 0 and each file is a sorted run. The behavior will be the same as initial universal compaction, so it can be used as a backward compatible mode.

## Compaction Picking Algorithm
Assuming we have sorted runs
```
    R1, R2, R3, ..., Rn
```
where R1 containing data of most recent updates to the DB and Rn containing the data of oldest updates of the DB. Note it is sorted by age of the data inside the sorted run, not the sorted run itself. According to this sorting order, after compaction, the output sorted run is always placed into the place where the inputs were.

How is all compactions are picked up:

#### Precondition: n >= options.level0_file_num_compaction_trigger
Unless number of sorted runs reaches this threshold, no compaction will be triggered at all.

(Note although the option name uses word "file", the trigger is for "sorted run" for historical reason. For the names of all options mentioned below, "file" also means sorted run for the same reason.)

If pre-condition is satisfied, there are three conditions. Each of them can trigger a compaction:

#### 1. Compaction Triggered by Space Amplification
If the estimated _size amplification ratio_ is larger than options.compaction_options_universal.max_size_amplification_percent / 100, all files will be compacted to one single sorted run.

Here is how _size amplification ratio_ is estimated:

```
    size amplification ratio = (size(R1) + size(R2) + ... size(Rn-1)) / size(Rn)
```
Please note, size of Rn is not included, which means 0 is the perfect size amplification and 100 means DB size is double the space of live data, and 200 means triple.

The reason we estimate size amplification in this way: in a stable sized DB, incoming rate of deletion should be similar to rate of insertion, which means for any of the sorted runs except Rn should include similar number of insertion and deletion. After applying R1, R2 ... Rn-1, to Rn, the size effects of them will cancel each other, so the output should also be size of Rn, which is the size of live data, which is used as the base of size amplification.

If options.compaction_options_universal.max_size_amplification_percent = 25, which means we will keep total space of DB less than 125% of total size of live data. Let's use this value in an example. Assuming compaction is only triggered by space amplification, options.level0_file_num_compaction_trigger = 1, file size after each mem table flush is always 1, and compacted size always equals to total input sizes. After two flushes, we have two files size of 1, while 1/1 > 25% so we'll need to do a full compaction:

```
1 1  =>  2
```

After another mem table flush we have

```
1 2   =>  3
```

Which again triggers a full compaction becasue 1/2 > 25%. And in the same way:

```
1 3  =>  4
```

But after another flush, the compaction is not triggered:

```
1 4
```

Because 1/4 <= 25%. Another mem table flush will trigger another compaction:

```
1 1 4  =>  6
```

Because (1+1) / 4 > 25%.

And it keeps going like this:

```
1
1 1  =>  2
1 2  =>  3
1 3  =>  4
1 4
1 1 4  =>  6
1 6
1 1 6  =>  8
1 8
1 1 8
1 1 1 8  =>  11
1 11
1 1 11
1 1 1 11  =>  14
1 14
1 1 14
1 1 1 14
1 1 1 1 14  =>  18
```

#### 2. Compaction Triggered by Individual Size Ratio
We calculated a value of _size ratio trigger_ as

```
     size_ratio_trigger = 1 + options.compaction_options_universal.size_ratio / 100
```
Usually options.compaction_options_universal.size_ratio is close to 0 so _size ratio trigger_ is close to 1.

We start from R1, if size(R2) / size(R1) < _size ratio trigger_, then (R1, R2) are qualified to be compacted together. We continue from here to determine whether R3 can be added too. If size(R1 + R2) / size(R3) < _size ratio trigger_, we would include (R1, R2, R3). Then we do the same for F4. We keep comparing total existing size to the next sorted run until the _size ratio trigger_ condition doesn't hold any more.

Here is an example to make it easier to understand. Assuming options.compaction_options_universal.size_ratio = 0, total mem table flush size is always 1, compacted size always equals to total input sizes, compaction is only triggered by space amplification and options.level0_file_num_compaction_trigger = 1. Now we start with only one file with size 1. After another mem table flush, we have two files size of 1, which triggers a compaction:

```
1 1  =>  2
```

After another mem table flush,

```
1 2  (no compaction triggered)
```

which doesn't qualify a flush because 1/2 < 1. But another mem table flush will trigger a compaction of all the files:

```
1 1 2  =>  4
```

This is because 1/1 >=1 and (1+1) / 2 >= 1.

The compaction will keep working like this:

```
1 1  =>  2
1 2  (no compaction triggered)
1 1 2  =>  4
1 4  (no compaction triggered)
1 1 4  =>  2 4
1 2 4  (no compaction triggered)
1 1 2 4 => 8
1 8  (no compaction triggered)
1 1 8  =>  2 8
1 2 8  (no compaction triggered)
1 1 2 8  =>  4 8
1 4 8  (no compaction triggered)
1 1 4 8  =>  2 4 8
1 2 4 8  (no compaction triggered)
1 1 2 4 8  =>  16
1 16  (no compaction triggered)
......
```

Compaction is only triggered when number of input sorted runs would be at least options.compaction_options_universal.min_merge_width and number of sorted runs as inputs will be capped as no more than  options.compaction_options_universal.max_merge_width.

#### 3. Compaction Triggered by number of sorted runs
If for every time we try to schedule a compaction, neither of 1 and 2 are triggered, we will try to compaction whatever sorted runs from R1, R2..., so that after the compaction the total number of sorted runs is not greater than options.level0_file_num_compaction_trigger + 1. If we need to compact more than options.compaction_options_universal.max_merge_width number of sorted runs, we cap it to options.compaction_options_universal.max_merge_width.

"Try to schedule" I mentioned below will happen when after flushing a memtable, finished a compaction. Sometimes duplicated attempts are scheduled.

See [Universal Style Compaction Example](https://github.com/facebook/rocksdb/wiki/Universal-Style-Compaction-Example) as an example of how output sorted run sizes look like for a common setting.

Parallel compactions are possible if options.max_background_compactions > 1. Same as all other compaction styles, parallel compactions will not work on the same sorted run.
 
## Subcompaction
Subcompaction is supported in universal compaction. If the output level of a compaction is not "level" 0, we will try to range partitioned the inputs and use number of threads of `options.max_subcompaction` to compact them in parallel. It will help with the problem that full compaction of universal compaction takes too long.

## Options to Tune
Following are options affecting universal compactions:
* options.compaction_options_universal: various options mentioned above
* options.level0_file_num_compaction_trigger the triggering condition of any compaction. It also means after all compactions' finishing, number of sorted runs will be under options.level0_file_num_compaction_trigger+1
* options.level0_slowdown_writes_trigger: if number of sorted runs exceed this value, writes will be artificially slowed down.
* options.level0_stop_writes_trigger: if number of sorted runs exceed this value, writes will stop until compaction finishes and number of sorted runs turn under this threshold.
* options.num_levels: if this value is 1, all sorted runs will be stored as level 0 files. Otherwise, we will try to fill non-zero levels as much as possible. The larger num_levels is, the less likely we will have large files on level 0.
* options.target_file_size_base: effective if options.num_levels > 1. Files of levels other than level 0 will be cut to file size not larger than this threshold.
* options.target_file_size_multiplier: it is effective, but we don't know a way to use this option in universal compaction that makes sense. So we don't recommend you to tune it.

Following options **DO NOT** affect universal compactions:
* options.max_bytes_for_level_base: only for level-based compaction
* options.level_compaction_dynamic_level_bytes: only for level-based compaction
* options.max_bytes_for_level_multiplier and options.max_bytes_for_level_multiplier_additional: only for level-based compaction
* options.expanded_compaction_factor: only for level-based compactions
* options.source_compaction_factor: only for level-based compactions
* options.max_grandparent_overlap_factor: only for level-based compactions
* options.soft_rate_limit and options.hard_rate_limit: deprecated
* options.hard_pending_compaction_bytes_limit: only used for level-based compaction
* options.compaction_pri: only supported in level-based compaction
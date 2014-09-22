## Overview
Universal Compaction Style is a compaction style, targeting the use cases requiring lower write amplification, trading off read amplification and space amplification.

When using this compaction style, all the SST files are put in L0. Data generated during a time range is stored in one SST file. Different SST files never overlap on their time ranges. Compaction can only happen among two or more files of adjacent time ranges. The output is a single file whose time range is the combination of input files. After any compaction, the condition that SST files never overlap on their time ranges still holds. 

## Compaction Picking Algorithm
Assuming we have file
```
    F1, F2, F3, ..., Fn
```
where F1 containing data of most recent updates to the DB and Fn containing the data of oldest updates of the DB. Note it is sorted by age of the data inside the file, not the file itself. According to this sorting order, after compaction, the output file is always placed into the place where the inputs were.

How is all compactions are picked up:

#### Precondition: n >= options.level0_file_num_compaction_trigger
Unless number of files reaches this threshold, no compaction will be triggered at all.

If pre-condition is satisfied, there are three conditions. Each of them can trigger a compaction:

#### 1. Compaction Triggered by Space Amplification
If the estimated _size amplification ratio_ is larger than options.compaction_options_universal.max_size_amplification_percent / 100, all files will be compacted to one single file.

Here is how _size amplification ratio_ is estimated:

```
    size amplification ratio = (size(F1) + size(F2) + ... size(Fn-1)) / size(Fn)
```
Please note, size of Fn is not included, which means 0 is the perfect size amplification and 100 means DB size is double the space of live data, and 200 means triple.

The reason we estimate size amplification in this way: in a stable sized DB, incoming rate of deletion should be similar to rate of insertion, which means for any of the files except Fn should include similar number of insertion and deletion. After applying F1, F2 ... Fn-1, to Fn, the size effects of them will cancel each other, so the output should also be size of Fn, which is the size of live data, which is used as the base of size amplification.

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

We start from F1, if size(F2) / size(F1) < _size ratio trigger_, then (F1, F2) are qualified to be compacted together. We continue from here to determine whether F3 can be added too. If size(F1 + F2) / size(F3) < _size ratio trigger_, we would include (F1, F2, F3). Then we do the same for F4. We keep comparing total existing size to the next file until the _size ratio trigger_ condition doesn't hold any more.

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

Compaction is only triggered when number of input files would be at least options.compaction_options_universal.min_merge_width and number of files as inputs will be capped as no more than  options.compaction_options_universal.max_merge_width.

#### 3. Compaction Triggered by number of files
If for every time we try to schedule a compaction, neither of 1 and 2 are triggered, we will try to compaction whatever files from F1, F2..., so that after the compaction the total number of files is not greater than options.level0_file_num_compaction_trigger + 1. If we need to compact more than options.compaction_options_universal.max_merge_width number of files, we cap it to options.compaction_options_universal.max_merge_width.

"Try to schedule" I mentioned below will happen when after flushing a memtable, finished a compaction. Sometimes duplicated attempts are scheduled.


Parallel compactions are possible if options.max_background_compactions > 1. Same as all other compaction styles, parallel compactions will not work on the same file.

 
This wiki page is not completed.
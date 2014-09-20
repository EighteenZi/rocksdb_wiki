## Overview
Universal Compaction Style is a compaction style, targeting the use case that requires lower write amplification, trading off read amplification and space amplification.

When using this compaction style, all the SST files are put in L0. Data generated during a time range is stored in one SST file. Different SST files never overlap on their time ranges. Compaction can only happen among two or more files of adjacent time ranges. The output is a single file whose time range is the combination of input files. After any compaction, the condition that SST files never overlap on their time ranges still holds. 

## Compaction Picking Algorithm
Assuming we have file
```
    F1, F2, F3, ..., Fn
```
where F1 containing the newest data and Fn containing the oldest.

Here are how a compaction is triggered:

* Precondition: n >= options.level0_file_num_compaction_trigger

There are three conditions, each of them can trigger a condition:

#### 1. Compaction Triggered by Space Amplification
If the estimated _size amplification ratio_ is larger than options.compaction_options_universal.max_size_amplification_percent / 100, all files will be compacted to one single file.

Here is how _size amplification ratio_ is estimated:

```
    size amplification ratio = (size(F1) + size(F2) + ... size(Fn-1)) / size(Fn)
```
Please note, size of Fn is not included, which means 0 is the perfect size amplification and 1 means doubling the space.

The reason we estimate size amplification in this way: in a stable sized DB, incoming rate of deletion should be similar to rate of insertion, which means for any of the files except Fn should include similar number of insertion and deletion. After applying F1, F2 ... Fn-1, to Fn, the size effects of them will cancel each other, so the output should also be size of Fn, which is the size of live data, which is used as the base of size amplification.

#### 2. Compaction Triggered by Individual Size Ratio
We calculated a value of _size ratio trigger_ as

```
     size_ratio_trigger = 1 + options.compaction_options_universal.size_ratio / 100
```
Usually options.compaction_options_universal.size_ratio is close to 0 so _size ratio trigger_ is close to 1.

We start from F1, if size(F2) / size(F1) < _size ratio trigger_, then (F1, F2) are qualified to be compacted together. We continue from here to determine whether F3 can be added too. If size(F1 + F2) / size(F3) < _size ratio trigger_, we would include (F1, F2, F3). Then we do the same for F4. We keep comparing total existing size to the next file until the _size ratio trigger_ condition doesn't hold any more.

Compaction is only triggered when number of input files would be at least options.compaction_options_universal.min_merge_width and number of files as inputs will be capped as no more than  options.compaction_options_universal.max_merge_width.

#### 3. Compaction Triggered by number of files
If for every time we try to schedule a compaction, neither of 1 and 2 are triggered, we will try to compaction whatever files from F1, F2..., so that after the compaction the total number of files is not greater than options.level0_file_num_compaction_trigger + 1. If we need to compact more than options.compaction_options_universal.max_merge_width number of files, we cap it to options.compaction_options_universal.max_merge_width.

"Try to schedule" I mentioned below will happen when after flushing a memtable, finished a compaction. Sometimes duplicated attempts are scheduled.


Parallel compactions are possible if options.max_background_compactions > 1. Same as all other compaction styles, parallel compactions will not work on the same file.

 
This wiki page is not completed.
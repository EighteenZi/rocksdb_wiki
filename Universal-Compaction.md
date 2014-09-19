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
Please note, size of Fn-1 is not included, which means 0 is the perfect size amplification and 1 means doubling the space.

The reason we estimate size amplification in this way: in a stable sized DB, incoming rate of deletion should be similar to rate of insertion, which means for any of the files except Fn should include similar number of insertion and deletion. After applying F1, F2 ... Fn-1, to Fn, the size effects of them will cancel each other, so the output should also be size of Fn, which is the size of live data, which is used as the base of size amplification.

This wiki page is not completed.
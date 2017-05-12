You can read more on Compactions here:
<a href="https://github.com/facebook/rocksdb/wiki/RocksDB-Basics#multi-threaded-compactions">Multi-threaded compactions</a>

Here we give overview of the options that impact behavior of Compactions:
<ul>

<li><code>Options::compaction_style</code> - RocksDB currently supports two compaction algorithms - Universal style and Level style. This option switches between the two. Can be kCompactionStyleUniversal or kCompactionStyleLevel. If this is kCompactionStyleUniversal, then you can configure universal style parameters with <code>Options::compaction_options_universal</code>.

<li><code>Options::disable_auto_compactions</code> - Disable automatic compactions. Manual compactions can still be issued on this database.

<li><code>Options::compaction_filter</code> - Allows an application to modify/delete a key-value during background compaction. The client must provide compaction_filter_factory if it requires a new compaction filter to be used for different compaction processes. Client should specify only one of filter or factory.

<li><code>Options::compaction_filter_factory</code> - a factory that provides compaction filter objects which allow an application to modify/delete a key-value during background compaction.
</ul>

Other options impacting performance of compactions and when they get triggered are:
<ul>

<li> <code>Options::access_hint_on_compaction_start</code> - Specify the file access pattern once a compaction is started. It will be applied to all input files of a compaction. Default: NORMAL

<li> <code>Options::level0_file_num_compaction_trigger</code> - Number of files to trigger level-0 compaction. A negative value means that level-0 compaction will not be triggered by number of files at all.

<li> <code>Options::target_file_size_base</code> and <code>Options::target_file_size_multiplier</code> - Target file size for compaction. target_file_size_base is per-file size for level-1. Target file size for level L can be calculated by target_file_size_base * (target_file_size_multiplier ^ (L-1)) For example, if target_file_size_base is 2MB and target_file_size_multiplier is 10, then each file on level-1 will be 2MB, and each file on level 2 will be 20MB, and each file on level-3 will be 200MB. Default target_file_size_base is 2MB and default target_file_size_multiplier is 1.

<li> <code>Options::max_compaction_bytes</code> - Maximum number of bytes in all compacted files. We avoid expanding the lower level file set of a compaction if it would make the total compaction cover more than this amount.

<li> <code>Options::max_background_compactions</code> - Maximum number of concurrent background jobs, submitted to the default LOW priority thread pool
<li> <code>Options::compaction_readahead_size</code> - If non-zero, we perform bigger reads when doing compaction. If you're running RocksDB on spinning disks, you should set this to at least 2MB. We enforce it to be 2MB if you don't set it with direct I/O. 
</ul>


You can learn more about all of those options in <code>rocksdb/options.h</code>

##  Leveled style compaction

See [[Leveled Compaction]].

##  Universal style compaction

For description about universal style compaction, see [Universal compaction style](https://github.com/facebook/rocksdb/wiki/Universal-Compaction)

If you're using Universal style compaction, there is an object <code>CompactionOptionsUniversal</code> that hold all the different options for that compaction. The exact definition is in <code>rocksdb/universal_compaction.h</code> and you can set it in <code>Options::compaction_options_universal</code>. Here we give short overview of options in <code>CompactionOptionsUniversal</code>: <ul>

<li> <code>CompactionOptionsUniversal::size_ratio</code> - Percentage flexibility while comparing file size. If the candidate file(s) size is 1% smaller than the next file's size, then include next file into this candidate set. Default: 1

<li> <code>CompactionOptionsUniversal::min_merge_width</code> - The minimum number of files in a single compaction run. Default: 2

<li> <code>CompactionOptionsUniversal::max_merge_width</code> - The maximum number of files in a single compaction run. Default: UINT_MAX

<li> <code>CompactionOptionsUniversal::max_size_amplification_percent</code> - The size amplification is defined as the amount (in percentage) of additional storage needed to store a single byte of data in the database. For example, a size amplification of 2% means that a database that contains 100 bytes of user-data may occupy upto 102 bytes of physical storage. By this definition, a fully compacted database has a size amplification of 0%. Rocksdb uses the following heuristic to calculate size amplification: it assumes that all files excluding the earliest file contribute to the size amplification. Default: 200, which means that a 100 byte database could require upto 300 bytes of storage.

<li> <code>CompactionOptionsUniversal::compression_size_percent</code> - If this option is set to be -1 (the default value), all the output files will follow compression type specified. If this option is not negative, we will try to make sure compressed size is just above this value. In normal cases, at least this percentage of data will be compressed. When we are compacting to a new file, here is the criteria whether it needs to be compressed: assuming here are the list of files sorted by generation time: [ A1...An B1...Bm C1...Ct ], where A1 is the newest and Ct is the oldest, and we are going to compact B1...Bm, we calculate the total size of all the files as total_size, as well as the total size of C1...Ct as total_C, the compaction output file will be compressed iff total_C / total_size < this percentage

<li> <code>CompactionOptionsUniversal::stop_style</code> - The algorithm used to stop picking files into a single compaction run. Can be kCompactionStopStyleSimilarSize (pick files of similar size) or kCompactionStopStyleTotalSize (total size of picked files > next file).

Default: kCompactionStopStyleTotalSize
</ul>

## FIFO Compaction Style

See [[FIFO compaction style]]

## Thread pools

Compactions are executed in thread pools. See [[Thread Pool]].
FIFO compaction style is the simplest compaction strategy. It is suited for keeping event log data with very low overhead (query log for example).

In FIFO compaction, all files are in level 0. When total size of the data exceeds configured size (CompactionOptionsFIFO::max_table_files_size), we delete the oldest table file. This means that write amplification of data is always 1 (in addition to WAL write amplification).

Please use FIFO compaction style with caution. It's still an experimental feature not tested in production. It also doesn't honor database snapshots (yet).

## Future work

Currently, the idea of FIFO compaction is to provide size-bounded data storage with low overhead. However, there are some ideas how to also make it good for storage of ordered event logs with extremely high performance. For more discussion on that topic see: https://reviews.facebook.net/D18765
**# Introduction**
  -- from leveldb
  -- rocksdb
  -- kv store
  -- fb engg team

**# Assumptions and Goals**
  -- good for fast storage (flash)
  -- good for random reads
  -- tradeoff between read ampl, write ampl
  -- range scans and point lookups
  -- production settting needs configurable param
     num levels, size of files, size of dats in a level, etc

**# Disk Format**
  .sst files for data
  .log files for trasactions
  manifest_file for database versions
  LOG* for server information logs

**# Read write apis**
  * Puts
  -- batch puts
  -- disable wal
  -- async puts, fsync and fdatasync
  -- Iterators and snapshots
  -- batch commit to transaction log and manifest updates
  -- binary search for overlapping files for every level
  checksums for reads (default false)
  hardware assists
  bloom filters
  shared block cache
  ReadOnly mode

**# Compactions**
  -- multi-threaded
  -- thread pool per environment
  -- priority queues for merge sort
  -- user defined hook for implementing ttl, sanity checks, etc
  -- avoid compression for two levels, snappy, bzip, zlib

**# Incremental Backups**
  GetLiveFiles
  GetUpdatesSince
    -- wals are archived

**# Environments**
  posix (production)
  hdfs environment (prototype)

**# Tools and Tests**
  sst_dump
  manifest_dump
  compact database, change number of levels
  stress test

**# java api**

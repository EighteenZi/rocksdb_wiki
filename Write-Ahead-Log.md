## Overview

Every update to RocksDB is written to two places: 1) an in-memory data structure called memtable and 2) write ahead log(WAL) on disk. In the event of a failure, write ahead log can be used to completely recover the data in the memtable, which is necessary to restore the database to the original state. 
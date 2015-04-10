## Overview

Repairer does best effort recovery to recover as much data as possible after a disaster without compromising consistency. It does not guarantee bringing the database to a time consistent state.

## Repair Process

Repair process is broken into 4 phase:
* Find files
* Convert logs to tables
* Extract metadata
* Write Descriptor

#### Find files

The repairer goes through all the files in the directory, and classifies them based on their file name. Any file that cannot be identified by name will be ignored.

#### Convert logs to table

Every log file that is active is replayed. All sections of the file where the checksum does not match is skipped over. We intentionally give preference to data consistency.

#### Extract metadata

We scan every table to compute

* smallest/largest for the table
* largest sequence number in the table

If we are unable to scan the file, then we ignore the table.

#### Write Descriptor

We generate descriptor contents:

* log number is set to zero
* next-file-number is set to 1 + largest file number we found
* last-sequence-number is set to largest sequence# found across all tables 
* compaction pointers are cleared
* every table file is added at level 0

#### Possible optimization

1. Compute total size and use to pick appropriate max-level M
2. Sort tables by largest sequence# in the table
3. For each table: if it overlaps earlier table, place in level-0, else place in level-M.
4. We can provide options for time consistent recovery and unsafe recovery (ignore checksum failure when applicable)
5. Store per-table metadata (smallest, largest, largest-seq#, ...) in the table's meta section to speed up ScanTable.
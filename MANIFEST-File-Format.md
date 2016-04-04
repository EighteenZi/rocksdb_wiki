## Overview

RocksDB is file system and storage medium agnostic. File system metadata operations like create, delete, rename etc are not atomic, and are susceptible to inconsistencies in the event of system failure. Even with journaling turned on, file systems do not guarantee consistency on unclean restart (metadata/data ordering can be configured incorrectly or storage controller may have caching enabled). The POSIX file system does not support atomic batching of operations either. 

RocksDB has a built-in mechanism to overcome these limitations of POSIX file system by keeping a transactional log of file system metadata state called the MANIFEST. MANIFEST is used to restore the system to the latest know consistent state on a restart.    

## Terminology

"MANIFEST" refers to the system that keeps track of file system metadata operations in a transactional log
"Manifest log" refers to an individual log file that contains file system metadata state/edits
"CURRENT" refers to the latest manifest log 

## How does it work ?

MANIFEST is a transactional log of the file system state. MANIFEST consists of - manifest log files and latest manifest file pointer. Manifest logs are rolling log files named MANIFEST-<seq number>. The sequence number is always increasing. CURRENT is a special file that identifies the latest manifest log file.

On system start, the latest manifest log contains the state of the file system. Any subsequent change to file system state is logged to the manifest log file. When a manifest log file exceeds a certain size, a new manifest log file is created with the snapshot of the file system state. The latest manifest file pointer is updated and the file system is synced. Upon successful update to CURRENT file, the redundant manifest logs are purged. 

MANIFEST = { CURRENT, MANIFEST-<seq-no>* } 
CURRENT = File pointer to the latest manifest log
MANIFEST-<seq no> = Contains snapshot of filesystem state and subsequent modifications

## Version Edit

A certain state of RocksDB files layout at any given time is referred to as a version snapshot. Any modification to the version snapshot is considered a version edit.

version-edit = Simple file system metadata operation

A snapshot is created by joining a sequence of version-edits. Essentially, a manifest log file is a sequence of version edits.

version-snapshot = { version-edit* }

Manifest log file is a log of version edits.

manifest-log-file = { version-snapshot, version-edit* } = { version-edit* }


## Version Edit Disk Layout

Version edits are a sequence of edit records. The records are identified by the edit identification number. The general format of a version edit is :

+-------------+------ ......... ----------+
| Record ID   | Variable size record data |
+-------------+------ .......... ---------+

The different types of edit records   



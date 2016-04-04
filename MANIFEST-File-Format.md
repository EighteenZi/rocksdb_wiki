## Overview

RocksDB is file system and storage medium agnostic. File system metadata operations like create, delete, rename etc are not atomic, and are susceptible to inconsistencies in the event of system failure. Even with journaling turned on, file systems do not guarantee consistency on unclean restart (metadata/data ordering can be configured incorrectly or storage controller may have caching enabled). The POSIX file system does not support atomic batching of operations either. 

RocksDB has a built-in mechanism to overcome these limitations of POSIX file system by keeping a transactional log of file system metadata state called the MANIFEST. MANIFEST is used to restore the system to the latest know consistent state on a restart.    

## Terminology

```
"MANIFEST" refers to the system that keeps track of file system metadata operations in a transactional log
"Manifest log" refers to an individual log file that contains file system metadata state/edits
"CURRENT" refers to the latest manifest log 
```

## How does it work ?

MANIFEST is a transactional log of the file system state. MANIFEST consists of - manifest log files and latest manifest file pointer. Manifest logs are rolling log files named MANIFEST-<seq number>. The sequence number is always increasing. CURRENT is a special file that identifies the latest manifest log file.

On system start, the latest manifest log contains the state of the file system. Any subsequent change to file system state is logged to the manifest log file. When a manifest log file exceeds a certain size, a new manifest log file is created with the snapshot of the file system state. The latest manifest file pointer is updated and the file system is synced. Upon successful update to CURRENT file, the redundant manifest logs are purged. 

```
MANIFEST = { CURRENT, MANIFEST-<seq-no>* } 
CURRENT = File pointer to the latest manifest log
MANIFEST-<seq no> = Contains snapshot of filesystem state and subsequent modifications
```

## Version Edit

A certain state of RocksDB files layout at any given time is referred to as a version snapshot. Any modification to the version snapshot is considered a version edit.

```
version-edit = Simple file system metadata operation
```
A snapshot is created by joining a sequence of version-edits. Essentially, a manifest log file is a sequence of version edits.
```
version-snapshot = { version-edit* }
```
Manifest log file is a log of version edits.
```
manifest-log-file = { version-snapshot, version-edit* } = { version-edit* }
```

## Version Edit Layout

Manifest log is a sequence of edit records. The version edit record type is identified by the edit identification number. 

We use the following convention for marking the encoding type :

Simple data types
```
VarX   - Variable character encoding of intX
FixedX - Fixed character encoding of intX
```

Complex data types
```
String - Length prefixed string data
+-----------+--------------------+
| size (n)  | content of string  |
+-----------+--------------------+
|<- Var32 ->|<-- n            -->|
```

The general format of a version edit record is :
```
+-------------+------ ......... ----------+
| Record ID   | Variable size record data |
+-------------+------ .......... ---------+
<-- Var32 --->|<-- varies by type       -->
```

The different types of version edit records and their layout :

```
Comparator edit record:
+-------------+----------------+
| kComparator | String         |
+-------------+----------------+
<-- Var32 --->|

Log number edit record:
+-------------+----------------+
| kLogNumber  | log number     |
+-------------+----------------+
<-- Var32 --->|<-- Var64    -->|

Previous File Number edit record:
+------------------+----------------+
| kPrevFileNumber  | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|

Next File Number edit record:
+------------------+----------------+
| kNextFileNumber  | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|

Last Sequence Number edit record:
+------------------+----------------+
| kLastSequence    | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|

Max Column Family edit record:
+---------------------+----------------+
| kMaxColumnFamily    | log number     |
+---------------------+----------------+
<-- Var32         --->|<-- Var32    -->|

Deleted File edit record:
+-----------------+-------------+--------------+
| kDeletedFile    | level       | file number  |
+-----------------+-------------+--------------+
<-- Var32     --->|<-- Var32 -->|<-- Var64  -->|


New File edit record:
+--------------+-------------+--------------+------------+------------+----------+----------------+----------------+
| kNewFile4    | level       | file number  | file size  | smallest   | largest  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+------------+----------+----------------+----------------+


+--------------+-------------+--------------+------------+------------+----------+----------------+----------------+
| kNewFile2    | level       | file number  | file size  | smallest   | largest  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+------------+----------+----------------+----------------+


+--------------+-------------+--------------+-----------+------------+------------+----------+----------------+----------------+
| kNewFile3    | level       | file number  | Path ID   | file size  | smallest   | largest  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+-----------+------------+------------+----------+----------------+----------------+

```   



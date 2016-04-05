## Overview

RocksDB is file system and storage medium agnostic. File system operations are not atomic, and are susceptible to inconsistencies in the event of system failure. Even with journaling turned on, file systems do not guarantee consistency on unclean restart. POSIX file system does not support atomic batching of operations either. Hence, it is not possible to rely on metadata embedded in RocksDB datastore files to reconstruct the last consistent state of the RocksDB on restart. 

RocksDB has a built-in mechanism to overcome these limitations of POSIX file system by keeping a transactional log of RocksDB state changes called the MANIFEST. MANIFEST is used to restore RocksDB to the latest know consistent state on a restart.    

## Terminology

* _MANIFEST_ refers to the system that keeps track of RocksDB state changes in a transactional log
* _Manifest log_ refers to an individual log file that contains RocksDB state snapshot/edits
* _CURRENT_ refers to the latest manifest log 

## How does it work ?

MANIFEST is a transactional log of the RocksDB state changes. MANIFEST consists of - manifest log files and latest manifest file pointer. Manifest logs are rolling log files named MANIFEST-(seq number). The sequence number is always increasing. CURRENT is a special file that identifies the latest manifest log file.

On system (re)start, the latest manifest log contains the consistent state of RocksDB. Any subsequent change to RocksDB state is logged to the manifest log file. When a manifest log file exceeds a certain size, a new manifest log file is created with the snapshot of the RocksDB state. The latest manifest file pointer is updated and the file system is synced. Upon successful update to CURRENT file, the redundant manifest logs are purged. 

```
MANIFEST = { CURRENT, MANIFEST-<seq-no>* } 
CURRENT = File pointer to the latest manifest log
MANIFEST-<seq no> = Contains snapshot of RocksDB state and subsequent modifications
```

## Version Edit

A certain state of RocksDB at any given time is referred to as a version (aka snapshot). Any modification to the version is considered a version edit. A version (or RocksDB state snapshot) is constructed by joining a sequence of version-edits. Essentially, a manifest log file is a sequence of version edits.

```
version-edit      = Any RocksDB state change
version           = { version-edit* }
manifest-log-file = { version, version-edit* }
                  = { version-edit* }
```

## Version Edit Layout

Manifest log is a sequence of version edit records. The version edit record type is identified by the edit identification number. 

We use the following datatypes for encoding/decoding.

### Data Types

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

### Version Edit Record Format

Version edit records have the following format. The decoder identifies the record type using the record identification number.
```
+-------------+------ ......... ----------+
| Record ID   | Variable size record data |
+-------------+------ .......... ---------+
<-- Var32 --->|<-- varies by type       -->
```

### Version Edit Record Types and Layout

There are variety of edit record corresponding to different state changes of RocksDB.

Comparator edit record:
```
Captures the comparator name

+-------------+----------------+
| kComparator | data           |
+-------------+----------------+
<-- Var32 --->|<-- String   -->|
```

Log number edit record:
```
Lates WAL log file number

+-------------+----------------+
| kLogNumber  | log number     |
+-------------+----------------+
<-- Var32 --->|<-- Var64    -->|
```

Previous File Number edit record:
```
Previous manifest file number

+------------------+----------------+
| kPrevFileNumber  | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|
```

Next File Number edit record:
```
Next manifest file number

+------------------+----------------+
| kNextFileNumber  | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|
```

Last Sequence Number edit record:
```
Last sequence number of RocksDB

+------------------+----------------+
| kLastSequence    | log number     |
+------------------+----------------+
<-- Var32      --->|<-- Var64    -->|
```

Max Column Family edit record:
```
Adjust the maximum number of family columns allowed.

+---------------------+----------------+
| kMaxColumnFamily    | log number     |
+---------------------+----------------+
<-- Var32         --->|<-- Var32    -->|
```

Deleted File edit record:
```
Mark a file as deleted from database.

+-----------------+-------------+--------------+
| kDeletedFile    | level       | file number  |
+-----------------+-------------+--------------+
<-- Var32     --->|<-- Var32 -->|<-- Var64  -->|
```

New File edit record:

Mark a file as newly added to the database and provide RocksDB meta information.

* File edit record with compaction information
```
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
| kNewFile4    | level       | file number  | file size  | smallest_key   | largest_key  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
|<-- var32  -->|<-- var32 -->|<-- var64  -->|<-  var64 ->|<-- String   -->|<-- String -->|<-- var64    -->|<-- var64    -->|

+-----------+---------------+-------+------------------+-------+--------------+
|kPathID ---| Path size(n)  | path  | kNeedCompaction  | 1     | value (0/1)  |
+-----------+---------------+-------+------------------+-------+--------------+
<- var32  ->|<-- var32   -->|<- n ->|<-- var32      -->|<- 1 ->|<-- 1      -->|
```

* File edit record backward compatible
```
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
| kNewFile2    | level       | file number  | file size  | smallest_key   | largest_key  | smallest_seqno | largest_seq_no |
+--------------+-------------+--------------+------------+----------------+--------------+----------------+----------------+
<-- var32   -->|<-- var32 -->|<-- var64  -->|<-  var64 ->|<-- String   -->|<-- String -->|<-- var64    -->|<-- var64    -->|
```

* File edit record with path information
```
+--------------+-------------+--------------+-------------+-------------+----------------+--------------+
| kNewFile3    | level       | file number  | Path ID     | file size   | smallest_key   | largest_key  |
+--------------+-------------+--------------+-------------+-------------+----------------+--------------+
|<-- var32  -->|<-- var32 -->|<-- var64  -->|<-- var32 -->|<-- var64 -->|<-- String   -->|<-- String -->|
+----------------+----------------+
| smallest_seqno | largest_seq_no |
+----------------+----------------+
<-- var64     -->|<-- var64    -->|
```   

Column family status edit record:
```
Note the status of column family feature (enabled/disabled)

+------------------+----------------+
| kColumnFamily    | 0/1            |
+------------------+----------------+
<-- Var32      --->|<-- Var32    -->|
```

Column family add edit record:
```
Add a column family

+---------------------+----------------+
| kColumnFamilyAdd    | cf name        |
+---------------------+----------------+
<-- Var32         --->|<-- String   -->|
```

Column family drop edit record:
```
Drop all column family

+---------------------+
| kColumnFamilyDrop   |
+---------------------+
<-- Var32         --->|
```

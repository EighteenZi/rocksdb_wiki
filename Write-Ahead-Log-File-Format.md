## Overview

Write ahead log (WAL) serializes memtable operations to persistent medium as log files. In the event of a failure, WAL files can be used to recover the database to its consistent state, by reconstructing the memtable from the logs. When a memtable is flushed out to persistent medium safely, the corresponding WAL log(s) become obsolete and are achieved. Eventually the archived logs are purged from disk after a certain period of time.

## WAL Manager

WAL files are generated with increasing sequence number in the WAL directory. In order to reconstruct the state of the database, these files are read in the order of sequence number. WAL manager provides the abstraction for reading the WAL files as a single unit. Internally, it opens and reads the files using Reader or Writer abstraction.

## Reader/Writer

Writer provides an abstraction for appending log records to a log file. The medium specific internal details are handled by WriteableFile interface. Similarly, Reader provides an abstraction for sequentially reading log records from the a log file. The internal medium specific details are handled by SequentialFile interface.

## Log File Format

Log file consists of a sequence of variable length records. Records are grouped by kBlockSize. If a certain record cannot fit into the leftover space, then the leftover space is padded with empty (null) data. The writer writes and the reader reads in chunks of kBlockSize.

```
       +-----+-------------+--+----+----------+------+-- ... ----+
 File  | r0  |        r1   |P | r2 |    r3    |  r4  |           |
       +-----+-------------+--+----+----------+------+-- ... ----+
       <--- kBlockSize ------>|<-- kBlockSize ------>|

  rn = variable size records
  P = Padding
```

### Record Format

The record layout format is as shown below.

```
+---------+-----------+-----------+--- ... ---+
|CRC (4B) | Size (2B) | Type (1B) | Payload   |
+---------+-----------+-----------+--- ... ---+

CRC = 32bit hash computed over the payload using CRC
Size = Length of the payload data
Type = Type of record
       (kZeroType, kFullType, kFirstType, kLastType, kMiddleType )
       The type is used to group a bunch of records together to represent
       blocks that are larger than kBlockSize
Payload = Byte stream as long as specified by the payload size
```
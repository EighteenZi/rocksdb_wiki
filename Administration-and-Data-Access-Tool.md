# Ldb Tool
The ldb command line tool offers multiple data access and database admin commands. Some examples are listed below. For more information, please consult the help message displayed when running ldb without any arguments and the unit tests in tools/ldb_test.py.

Example data access sequence:

```bash
    $./ldb --db=/tmp/test_db --create_if_missing put a1 b1
    OK 


    $ ./ldb --db=/tmp/test_db get a1
    b1
 
    $ ./ldb --db=/tmp/test_db get a2
    Failed: NotFound:

    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
 
    $ ./ldb --db=/tmp/test_db scan --hex
    0x6131 : 0x6231
 
    $ ./ldb --db=/tmp/test_db put --key_hex 0x6132 b2
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
    a2 : b2
 
    $ ./ldb --db=/tmp/test_db get --value_hex a2
    0x6232
 
    $ ./ldb --db=/tmp/test_db get --hex 0x6131
    0x6231
 
    $ ./ldb --db=/tmp/test_db batchput a3 b3 a4 b4
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    a1 : b1
    a2 : b2
    a3 : b3
    a4 : b4
 
    $ ./ldb --db=/tmp/test_db batchput "multiple words key" "multiple words value"
    OK
 
    $ ./ldb --db=/tmp/test_db scan
    Created bg thread 0x7f4a1dbff700
    a1 : b1
    a2 : b2
    a3 : b3
    a4 : b4
    multiple words key : multiple words value
```

To dump an existing leveldb database in HEX:
```bash
$ ./ldb --db=/tmp/test_db dump --hex > /tmp/dbdump
```

To load the dumped HEX format data to a new leveldb database:
```bash
$ cat /tmp/dbdump | ./ldb --db=/tmp/test_db_new load --hex --compression_type=bzip2 --block_size=65536 --create_if_missing --disable_wal
```

To compact an existing leveldb database:
```bash
$ ./ldb --db=/tmp/test_db_new compact --compression_type=bzip2 --block_size=65536
```

# SST dump tool
sst_dump tool can be used to gain insights about a specific SST file. There are multiple operations that sst_dump can execute on a SST file.

##### Dumping SST file blocks
```bash
./sst_dump --file=/path/to/sst/000829.sst --command=raw
``` 
This command will generate a txt file named /path/to/sst/000829_dump.txt
This file will contain all Index blocks and data blocks encoded in Hex. It will also contain information like table properties, footer details and meta index details

##### Printing entries in SST file
```bash
./sst_dump --file=/path/to/sst/000829.sst --command=scan --read_num=5
```
This command will print the first 5 keys in the SST file to the screen. the output may look like this
```
'Key1' @ 5: 1 => Value1
'Key2' @ 2: 1 => Value2
'Key3' @ 4: 1 => Value3
'Key4' @ 3: 1 => Value4
'Key5' @ 1: 1 => Value5
```
The output can be interpreted like this
```
'<key>' @ <sequence number>: <type> => <value>
```
Please notice that if your key have non-ascii characters it will be hard to print it on screen, in this case it's a good idea to use --output_hex like this
```bash
./sst_dump --file=/path/to/sst/000829.sst --command=scan --read_num=5 --output_hex
```

You can also specify where do you want to start reading from and where do you want to stop by using --from and --to like this
```bash
./sst_dump --file=/path/to/sst/000829.sst --command=scan --from="key2" --to="key4"
```

You can pass --from and --to using hexadecimal as well by using --input_key_hex
```bash
./sst_dump --file=/path/to/sst/000829.sst --command=scan --from="0x6B657932" --to="0x6B657934" --input_key_hex
```

##### Checking SST file
```bash
./sst_dump --file=/path/to/sst/000829.sst --command=check --verify_checksum
```
This command will Iterate over all entries in the SST file but wont print any thing except if it encountered a problem in the SST file. It will also verify the checksum

##### Printing SST file properties
```bash
./sst_dump --file=/path/to/sst/000829.sst --show_properties
```
This command will read the SST file properties and print them, output may look like this
```
from [] to []
Process /path/to/sst/000829.sst
Sst file format: block-based
Table Properties:
------------------------------
  # data blocks: 26541
  # entries: 2283572
  raw key size: 264639191
  raw average key size: 115.888262
  raw value size: 26378342
  raw average value size: 11.551351
  data block size: 67110160
  index block size: 3620969
  filter block size: 0
  (estimated) table size: 70731129
  filter policy name: N/A
  # deleted keys: 571272
```

##### Trying different compression algorithms
sst_dump can be used to check the size of the file under different compression algorithms.
```bash
./sst_dump --file=/path/to/sst/000829.sst --show_compression_sizes
```
By using --show_compression_sizes sst_dump will recreate the SST file in memory using different compression algorithms and report the size, output may look like this
```
from [] to []
Process /path/to/sst/000829.sst
Sst file format: block-based
Block Size: 16384
Compression: kNoCompression Size: 103974700
Compression: kSnappyCompression Size: 103906223
Compression: kZlibCompression Size: 80602892
Compression: kBZip2Compression Size: 76250777
Compression: kLZ4Compression Size: 103905572
Compression: kLZ4HCCompression Size: 97234828
Compression: kZSTDNotFinalCompression Size: 79821573
```

These files are created in memory and they are generated with block size of 16KB, the block size can be change by using --set_block_size
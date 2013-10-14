The ldb command line tool offers multiple data access and database admin commands. Some examples are listed below. For more information, please consult the help message displayed when running ldb without any arguments and the unit tests in tools/ldb_test.py.

Example data access sequence:

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


To dump an existing leveldb database in HEX:
`
$ ./ldb --db=/tmp/test_db dump --hex > /tmp/dbdump
`

To load the dumped HEX format data to a new leveldb database:
`
$ cat /tmp/dbdump | ./ldb --db=/tmp/test_db_new load --hex --compression_type=bzip2 --block_size=65536 --create_if_missing --disable_wal
`

To compact an existing leveldb database:
`
$ ./ldb --db=/tmp/test_db_new compact --compression_type=bzip2 --block_size=65536
`
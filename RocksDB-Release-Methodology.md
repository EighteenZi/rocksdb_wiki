# Time Based Releases

RocksDB will have time-based periodic releases. We typically try to make a release once every month. The release process involves compiling a 'release' build, running an extensive suite of tests and then making the 'release' on github.

The tests that we typically run are:

1. make check to run all unit tests
2. valgrind and asan checks
3. db_stress to validate data correctness
4. db_bench to validate performance results
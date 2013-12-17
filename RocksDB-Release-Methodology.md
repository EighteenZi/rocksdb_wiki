# Time Based Releases

RocksDB has time-based periodic releases. We typically try to make a release once every month. The release process involves compiling a 'release' build, running an extensive suite of tests and then making the 'release' on github.

The tests that we typically run are:

1. run all unit tests
2. valgrind and asan checks
3. db_stress to validate data correctness
4. db_bench to validate performance results

# New features and enhancements

The 'master' branch is used for code development. Most new features, performance improvements, new unit tests, etc are code reviewed via the open source link http://reviews.facebook.net, and once accepted, are committed to the 'master' branch. 

Regression tests (http://github.com/facebook/rocksdb/blob/master/build_tools/regression_build_test.sh) and valgrind tests (http://github.com/facebook/rocksdb/blob/master/build_tools/valgrind_test.sh) are run every night. Code coverage tools can be found at https://github.com/facebook/rocksdb/blob/master/coverage/coverage_test.sh.

# Large feature developments

Sometimes, a large feature development requires an iterative approach which could potentially take a few weeks or even months. These type of feature development happens on a branch, (e.g. 'performance' branch), this makes it easier to collaborate among multiple developers. Once the feature is complete and tested, then that branch is merged back to master.

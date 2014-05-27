# Time Based Releases

RocksDB has time-based periodic releases. We typically try to make a release once every month.

# Release version

Release version has three parts: MAJOR.MINOR.PATCH
* Major version number is bumped when there is an API change or disk format change. However, we usually try to maintain complete backwards compatibility for basic APIs. Basic users should always be able to upgrade to the newest RocksDB version without having to change any code. Advanced users might have to change some API calls when upgrading versions, but this is not common.
* Minor version number is bumped on normal release cycle. It usually comes with some new features and performance improvements.
* Patch version number is bumped on bug fixes. You should probably be using the release with the latest patch number.

See [RocksDB version macros](https://github.com/facebook/rocksdb/wiki/RocksDB-version-macros) for how to keep track with version you're using.

# Branches and tags

* The 'master' branch is used for code development. It should be somewhat stable and in usable state. It is not recommended for production use, though.
* When we release, we cut-off the branch from master. For example, release 3.0 will have a branch 3.0.fb. Once the branch is stable enough (bug fixes go to both master and the branch), we create a release tag with a name 'rocksdb-3.0.0'. While we develop version 3.1 in master branch, we maintain '3.0.fb' branch and push bug fixes there. We release 3.0.1, 3.0.2, etc. as tags on a '3.0.fb' branch with names 'rocksdb-3.0.X'. Once we release 3.1, we stop maintaining '3.0.fb' branch. We still don't have a concept of long term supported releases.
* Bigger features are developed in separate branches. Once the feature is proved to be stable it's merged into master and released with the next release. This does not apply to StackableDB framework. Addition of new StackableDB concepts does not touch RocksDB core code and can go directly to master branch.

# Testing

We use both Jenkins and Travis to continuously run tests on our master branch. We run:

1. All unit tests with `make check`.
2. All unit tests with valgrind -- `make valgrind_check`.
3. All unit tests with ASAN enabled -- `make asan_check`.
3. db_stress tests to verify data correctness. We run it in both normal and ASAN mode -- `make crash_test` and `make asan_crash_test`.
4. db_bench to detect any performance regressions -- `build_tools/regression_build_test.sh`

Every release is pushed to a big number of Facebook servers across lots of different services and configurations in the following week or two. This is the real test. We sometimes find issues in production that we fix by releasing new patches to the release branch. You can assume that releases that are two weeks old are very stable and throughly tested.
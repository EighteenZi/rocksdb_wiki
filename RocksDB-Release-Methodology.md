# Time Based Releases

RocksDB has time-based periodic releases. We typically try to make a release once every month.

# Release version

Release version has three parts: MAJOR.MINOR.PATCH
* Major version number is bumped when there is an API change or disk format change. However, we usually try to maintain complete backwards compatibility for basic APIs. Basic users should always be able to upgrade to the newest RocksDB version without having to change any code. Advanced users might have to change some API calls when upgrading versions, but this is not common.
* Minor version number is bumped on normal release cycle. It usually comes with some new features and performance improvements.
* Patch version number is bumped on bug fixes. You should probably be using the release with the latest patch number.

# Branches and tags

* The 'master' branch is used for code development. It should be somewhat stable and in usable state. It is not recommended for production use, though.
* When we release, we cut-off the branch from master. For example, release 3.0 will have a branch 3.0.fb. Once the branch is stable enough (bug fixes go to both master and the branch), we create a release tag with a name 'rocksdb-3.0'. While we develop version 3.1 in master branch, we maintain '3.0.fb' branch and push bug fixes there. We release 3.0.1, 3.0.2, etc. as tags on a '3.0.fb' branch with names 'rocksdb-3.0.X'. Once we release 3.1, we stop maintaining '3.0.fb' branch. We still don't have a concept of long term supported releases.
* Bigger features are developed in separate branches. Once the feature is proved to be stable it's merged into master and released with the next release. This does not apply to StackableDB framework. Addition of new StackableDB concepts does not touch RocksDB core code and can go directly to master branch.
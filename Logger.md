# Introduction

RocksDB supports a generalized message logging infrastructure. RocksDB caters to a variety of use cases -- from low power mobile systems to high end servers running distributed applications. The framework helps extent the message logging infrastructure as per the use case requirements. The mobile app might need a relatively simpler logging mechanism, compared to a server running mission critical application. It also provides a means to integrate RocksDB log messages with the embedded application logging infrastructure. 

# Exiting Logger Hierarchy

The [Logger](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/env.h#L663) class provides the interface definition for logging messages from RocksDB. 

The various implementations of Logger available are :

| Implementation        | Use           |
| ------------- |:-------------:| 
| NullLogger | /dev/null equivalent for logger| 
| StderrLogger| Pipes the messages to std::err equivalent| 
| HdfsLogger| Logs messages to HDFS|
| PosixLogger| Logs messages to POSIX file|
| AutoRollLogger| Automatically rolls files as they reach a certain size. Typically used for servers|
| WinLogger| Specialized logger for Windows OS|

# Writing your custom Logger

Users are encouraged to write their own logging infrastructure as per the use case by extending any one of the existing logger implementations.
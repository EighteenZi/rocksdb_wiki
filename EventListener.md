# EventListener
EventListener class contains a set of call-back functions that will be called when specific RocksDB event happens such as flush.  It can be used as a building block for developing custom features such as stats-collector or external compaction algorithm.

## How to use it?
In ColumnFamilyOptions, there's a variable called `listeners`, which allows developers to add custom EventListener to listen to the events of a specific rocksdb instance or a column family of a rocksdb instance.

    // A vector of EventListeners which call-back functions will be called
    // when specific RocksDB event happens.
    std::vector<std::shared_ptr<EventListener>> listeners;

To listening to a rocksdb instance or a column family of a rocksdb instance, it can be done by simply adding a custom EventListener to `ColumnFamilyOptions::listeners` and use that options to open a DB:

    // listen to a column family of a rocksdb instance
    ColumnFamilyOptions cf_options;
    ...
    cf_options.listeners.emplace_back(new MyListener());

To listen to multiple column families of a rocksdb instance, one can either use a separate Listener instance for each column family,

    // one listener for each column family
    for (size_t i = 0; i < cf_options.size(); ++i) {
      cf_options[i].listeners.emplace_back(new MyListener());
    }

or use same Listener instance for all column families:

    // one same listener for all column families.
    EventListener my_listener = new MyListener();
    for (size_t i = 0; i < cf_options.size(); ++i) {
      cf_options[i].listeners.emplace_back(my_listener);
    }

Note that in either case, unless specially specified in the documentation, all EventListener call-backs must be implemented in a thread-safe way even when an EventListener only listens to a single column family (For example, imagine the case where OnCompactionCompeted() could be called by multiple threads at the same time as a single column family might complete more than one compaction jobs at the same time.


## Developers Note
Note that call-back functions should not run for an extended period of time before the function returns, otherwise RocksDB may be blocked.  For example, it is not suggested to do DB::CompactFiles() (as it may run for a long while) or issue many of DB::Put() (as Put may be blocked in certain cases) in the same thread in the EventListener callback.  However, doing DB::CompactFiles() and DB::Put() in another thread is considered safe.

RocksDB provide the user with APIs that can be used to create SST files that can be ingested later. This can be useful if you have a use case that need to load the data quickly, but the process of creating the data can be done offline.

## Creating SST file
rocksdb::SstFileWriter can be used to create SST file. After creating a SstFileWriter object you can open a file, insert rows into it and finish.   

This is an example of how to create SST file in `/home/usr/file1.sst`

```cpp
Options options;
const ImmutableCFOptions ioptions(options);

SstFileWriter sst_file_writer(EnvOptions(), ioptions, options.comparator);
// Path to where we will write the SST file
std::string file_path = "/home/usr/file1.sst";

// Open the file for writing
Status s = sst_file_writer.Open(file_path);
if (!s.ok()) {
    printf("Error while opening file %s, Error: %s\n", file_path.c_str(),
           s.ToString().c_str());
    return 1;
}

// Insert rows into the SST file, note that inserted keys must be 
// strictly increasing (based on options.comparator)
for (...) {
  s = sst_file_writer.Add(key, value);
  if (!s.ok()) {
    printf("Error while adding Key: %s, Error: %s\n", key.c_str(),
           s.ToString().c_str());
    return 1;
  }
}

// Close the file
s = sst_file_writer.Finish();
if (!s.ok()) {
    printf("Error while finishing file %s, Error: %s\n", file_path.c_str(),
           s.ToString().c_str());
    return 1;
}
return 0;
```

Now we have our SST file located at `/home/usr/file1.sst`.

Please note that:  
*    ImmutableCFOptions passed to SstFileWriter will be used to figure out the table type, compression options, etc that will be used to create the SST file.
*    The Comparator that is passed to the SstFileWriter must be exactly the same as the Comparator used in the DB that this file will be ingested into.
*    Rows must be inserted in a strictly increasing order. 

You can learn more about the SstFileWriter by checking [include/rocksdb/sst_file_writer.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/sst_file_writer.h)
## Ingesting SST files
Ingesting an SST files is simple, all you need to do is to call DB::AddFile() and pass the file paths as a vector of `std::string`:
```cpp
std::string file_path1 = "/home/usr/file1.sst";
std::string file_path2 = "/home/usr/file2.sst";
Status s = db_->AddFile({file_path1, file_path2});
if (!s.ok()) {
  printf("Error while adding file %s and %s, Error %s\n",
         file_path1.c_str(), file_path2.c_str(), s.ToString().c_str());
  return 1;
}
```
Please notice there are some restrictions on the SST files that you can ingest into a DB:
* SST file must have been created using SstFileWriter.
* The Comparator used to create the SST file must match the Comparator used by the DB.
* Key ranges in the SST files don't overlap with each other or existing/deleted keys in the DB.
* No snapshots are being held while ingesting the files.

Note: The API `DB::AddFile(std::string file_path)` that accepts one file path is **deprecated** and should not be used. Please notice that when you need to ingest one file, do not call `DB:AddFile` with initialization list, but with an explicit vector because of the wrong implicit conversion from `std::initializer_list<std::string>` to `std::string`. 
```cpp
std::string file_path1 = "/home/usr/file1.sst";
db_->AddFile(std::vector<std::string>(1,file_path1));
db_->AddFile({file_path1}); // Error
```

You can learn more by checking DB::AddFile() in [include/rocksdb/db.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/db.h)

---
Do you have a use case that can benefit from using DB::AddFile() but the current restrictions are too tight ? Please let us know about your use case and what restrictions are blocking you. 
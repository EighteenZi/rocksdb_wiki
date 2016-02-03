# Creating and Ingesting SST files
RocksDB provide the user with APIs that can be used to create SST files that can be ingested later. This can be useful if you have a use case that need to load the data quickly, but the process of creating the data can be done offline.

## Creating SST file
rocksdb::SstFileWriter can be used to create SST file. After creating a SstFileWriter object you can open a file, insert rows into it and finish.   

This is an example of how to create SST file in `/home/usr/file1.sst`

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

Now we have our SST file located at `/home/usr/file1.sst`.

Please note that:  
*    ImmutableCFOptions passed to SstFileWriter will be used to figure out the table type, compression options, etc that will be used to create the SST file.
*    The Comparator that is passed to the SstFileWriter must be exactly the same as the Comparator used in the DB that this file will be ingested into.
*    Rows must be inserted in a strictly increasing order. 

You can learn more about the SstFileWriter by checking [include/rocksdb/sst_file_writer.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/sst_file_writer.h)
## Ingesting SST file
Ingesting an SST file is simple, all you need to do is to call DB::AddFile() and pass the file path

    std::string file_path = "/home/usr/file1.sst";
    Status s = db_->AddFile(file_path);
    if (!s.ok()) {
      printf("Error while adding file %s, Error %s\n", file_path.c_str(),
             s.ToString().c_str());
      return 1;
    }

Please notice there are some restrictions on the SST files that you can ingest into a DB:
* SST file must have been created using SstFileWriter.
* The Comparator used to create the SST file must match the Comparator used by the DB.
* Key range in the SST file dont overlap with existing keys or deleted keys in the DB.
* No snapshots are being held while ingesting the file.

You can learn more by checking DB::AddFile() in [include/rocksdb/db.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/db.h)

---
Do you have a use case that can benefit from using DB::AddFile() but the current restrictions are too tight ? Please let us know about your use case and what restrictions are blocking you. 
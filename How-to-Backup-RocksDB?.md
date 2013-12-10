In RocksDB, we have implemented an easy way to backup your DB. Here is a simple example:
 
    #include "rocksdb/db.h"
    #include "utilities/backupable_db.h"
    using namespace rocksdb;

    DB* db;
    DB::Open(Options(), "/tmp/rocksdb", &db);
    BackupableDB* backupable_db = new BackupableDB(db, BackupableDBOptions("/tmp/rocksdb_backup"));
    backupable_db->Put(...); // do your thing 
    backupable_db->CreateNewBackup();
    delete backupable_db; // no need to also delete db
  
This simple example will create a backup of your DB in "/tmp/rocksdb_backup". Restoring is also easy:

    RestoreBackupableDB* restore = new RestoreBackupableDB(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"));
    restore->RestoreDBFromLatestBackup("/tmp/rocksdb", "/tmp/rocksdb");
    delete restore;

You have restored the backup back to "/tmp/rocksdb". The second parameter is the location of log files (in some DBs they are different from DB directory, but usually it's the same).


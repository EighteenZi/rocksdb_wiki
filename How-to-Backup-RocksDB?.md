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

This code will restore the backup back to "/tmp/rocksdb". The second parameter is the location of log files (In some DBs they are different from DB directory, but usually they are the same. See Options::wal_dir for more info).

Backups are incremental. You can create a new backup with `CreateNewBackup()` and only the new data will be copied to backup directory. Once you have more backups saved, you can issue `GetBackupInfo()` call to get a list of all backups together with information on timestamp of the backup and the size (please note that sum of all backups' sizes is bigger than the actual size of the backup directory because some data is shared by multiple backups). Backups are identified by their always-increasing IDs. `GetBackupInfo()` is available both in `BackupableDB` and `RestoreBackupableDB`.

You can keep around only small fixed number of backups. To delete old backups, you can issue call `PurgeOldBackups(N)`, where N is how many backups you'd like to keep. All backups except the N newest ones will be deleted. You can also choose to delete arbitrary backup with call `DeleteBackup(id)`.

`RestoreDBFromLatestBackup()` will restore the DB from the latest consistent backup. An alternative is `RestoreDBFromBackup()` which takes a backup ID and restores that particular backup. Very important thing to note here: Let's say you have backups 1, 2, 3, 4. If you restore from backup 2 and start putting more data to your database, newly created backups might conflict with backups 3 and 4. There are two ways to solve that: (1) delete backups 3 and 4, or (2) store new backups in different backup directory.

### Advanced usage
Let's say you want to backup your DB to HDFS. There is an option in `BackupableDBOptions` to set `backup_env`, which will be used for all file I/O related to backup dir (writes when backuping, reads when restoring). If you set it to HDFS Env, all the backups will be stored in HDFS.

`BackupableDBOptions::info_log` is a Logger object that is used to print out LOG messages if not-nullptr.

If `BackupableDBOptions::sync` is true, we will sync data to disk after every file write, guaranteeing that backups will be consistent after a reboot or if machine crashes. Setting it to false will speed things up a bit, but some (newer) backups might be inconsistent. In most cases, everything should be fine, though.

If you set `BackupableDBOptions::destroy_old_data` to true, creating new `BackupableDB` will delete all the old backups in the backup directory.

`BackupableDB::CreateNewBackup()` method takes a parameter `flush_before_backup`, which is false by default. When `flush_before_backup` is true, `BackupableDB` will first issue a memtable flush and only then copy the DB files to the backup directory. Doing so will prevent log files from being copied to the backup directory (since flush will delete them). If `flush_before_backup` is false, backup will not issue flush before starting the backup. In that case, the backup will also include log files corresponding to live memtables. Backup will be consistent with current state of the database regardless of `flush_before_backup` parameter.


## Further reading
For the API details, see "include/utilities/backupable_db.h". For the implementation, see "utilities/backupable/backupable_db.cc".
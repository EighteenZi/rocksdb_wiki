In RocksDB, we have implemented an easy way to backup your DB. Here is a simple example:
 
```cpp
    #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    using namespace rocksdb;

    int main() {
        Options options;                                                                                  
        options.create_if_missing = true;                                                                 
        DB* db;
        Status s = DB::Open(options, "/tmp/rocksdb", &db);
        assert(s.ok());
        db->Put(...); // do your thing

        BackupEngine* backup_engine;
        s = BackupEngine::Open(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"), &backup_engine);
        assert(s.ok());
        backup_engine->CreateNewBackup(db);
        delete db;
        delete backup_engine;
    }
```

This simple example will create a backup of your DB in "/tmp/rocksdb_backup".

Restoring is also easy:

```cpp
    #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    using namespace rocksdb;

    int main() {
        BackupEngineReadOnly* backup_engine;
        Status s = BackupEngineReadOnly::Open(Env::Default(), BackupableDBOptions("/tmp/rocksdb_backup"), &backup_engine);
        assert(s.ok());
        backup_engine->RestoreDBFromLatestBackup("/tmp/rocksdb", "/tmp/rocksdb");
        delete backup_engine;
    }
```

This code will restore the backup back to "/tmp/rocksdb". The first parameter of `RestoreDBFromLatestBackup()` is the target DB directory. The second parameter is the target location of log files (in some DBs they are different from DB directory, but usually they are the same. See Options::wal_dir for more info).

Be aware, that backup engine's `Open()` takes time proportional to amount of backups. So if you have slow filesystem to backup (like HDFS), and you have a lot of backups, then restoring or creating further backups can take some time. That's why we recommend to limit the number of backups. Also we recommend to keep BackupEngine alive and not to recreate it every time you need to do a backup.

Backups are incremental as long as BackupableDBOptions::share_table_files is set. You can create a new backup with `CreateNewBackup()` and only the new data will be copied to backup directory (for more details on what gets copied, see [Under the hood](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F#under-the-hood)).

Checksums are always stored separately for any backed up file (including sst, log, and etc), and file size is stored in filename when Options::share_files_with_checksum is set. These attributes can be used to uniquely identify files when multiple databases share a single backup directory. Additionally, file size is used for consistency checking when `VerifyBackups()` is called -- we do not verify checksum, however, since it would require reading all the data.

Once you have some backups saved, you can issue `GetBackupInfo()` call to get a list of all backups together with information on timestamp of the backup and the size (please note that sum of all backups' sizes is bigger than the actual size of the backup directory because some data is shared by multiple backups). Backups are identified by their always-increasing IDs.

You probably want to keep around only small number of backups. To delete old backups, just call `PurgeOldBackups(N)`, where N is how many backups you'd like to keep. All backups except the N newest ones will be deleted. You can also choose to delete arbitrary backup with call `DeleteBackup(id)`.

`RestoreDBFromLatestBackup()` will restore the DB from the latest consistent backup. An alternative is `RestoreDBFromBackup()` which takes a backup ID and restores that particular backup. Checksum is calculated for any restored file and compared against the one stored during the backup time. If a checksum mismatch is detected, the restore process is aborted and `Status::Corruption` is returned. Very important thing to note here: Let's say you have backups 1, 2, 3, 4. If you restore from backup 2 and start writing more data to your database, newly created backup will delete old backups 3 and 4 and create new backup 3 on top of 2. 

### Advanced usage
Let's say you want to backup your DB to HDFS. There is an option in `BackupableDBOptions` to set `backup_env`, which will be used for all file I/O related to backup dir (writes when backuping, reads when restoring). If you set it to HDFS Env, all the backups will be stored in HDFS.

`BackupableDBOptions::info_log` is a Logger object that is used to print out LOG messages if not-nullptr.

If `BackupableDBOptions::sync` is true, we will sync data to disk after every file write, guaranteeing that backups will be consistent after a reboot or if machine crashes. Setting it to false will speed things up a bit, but some (newer) backups might be inconsistent. In most cases, everything should be fine, though.

If you set `BackupableDBOptions::destroy_old_data` to true, creating new `BackupEngine` will delete all the old backups in the backup directory.

`BackupEngine::CreateNewBackup()` method takes a parameter `flush_before_backup`, which is false by default. When `flush_before_backup` is true, `BackupEngine` will first issue a memtable flush and only then copy the DB files to the backup directory. Doing so will prevent log files from being copied to the backup directory (since flush will delete them). If `flush_before_backup` is false, backup will not issue flush before starting the backup. In that case, the backup will also include log files corresponding to live memtables. Backup will be consistent with current state of the database regardless of `flush_before_backup` parameter.

### Under the hood
When you call `BackupEngine::CreateNewBackup()`, it does the following:

1. Disable file deletions
2. Get live files (this includes table files, current and manifest file).
3. Copy live files to the backup directory. Since table files are immutable and filenames unique, we don't copy a table file that is already present in the backup directory. For example, if there is a file `00050.sst` already backed up and `GetLiveFiles()` returns `00050.sst`, we will not copy that file to the backup directory. However, checksum is calculated for all files regardless if a file needs to be copied or not. If a file is already present, the calculated checksum is compared against previously calculated checksum to make sure nothing crazy happened between backups. If a mismatch is detected, backup is aborted and the system is restored back to the state before `BackupEngine::CreateNewBackup()` is called. One thing to note is that a backup abortion could mean a corruption from a file in backup directory or the corresponding live file in current DB. Both manifest and current files are copied, since they are not immutable.
4. If `flush_before_backup` was set to false, we also need to copy log files to the backup directory. We call `GetSortedWalFiles()` and copy all live files to the backup directory.
5. Enable file deletions

Backup IDs are always increasing and we have a file `LATEST_BACKUP` that contains the ID of the latest backup. If we crash in middle of backing up, on a restart we will detect that there are newer backup files than `LATEST_BACKUP` claims there are. In that case, we will delete any backup newer than `LATEST_BACKUP` and clean up all the files since some of the table files might be corrupted. Having corrupted table files in the backup directory is dangerous because of our deduplication strategy.

### Further reading
For the API details, see `include/utilities/backupable_db.h`. For the implementation, see `utilities/backupable/backupable_db.cc`.
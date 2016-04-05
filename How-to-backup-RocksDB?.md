### Backup API

For the C++ API, see `include/rocksdb/utilities/backupable_db.h`. The key abstraction is the backup engine, which exposes simple interfaces to create backups, get info about backups, and restore from backup. There are two distinct representations of backup engines: (1) `BackupEngine` for creating new backups, and (2) `BackupEngineReadOnly` for restoring from backup. Either one can be used to get info about backups.

Be aware, that backup engine's `Open()` takes time proportional to the number of existing backups. So if you have slow filesystem to backup (like HDFS), and you have a lot of backups, then initializing the backup engine can take some time. We recommend to keep your backup engine alive and not to recreate it every time you need to do a backup or restore.

Also, we recommend to keep around only small number of backups. To delete old backups, just call `PurgeOldBackups(N)`, where N is how many backups you'd like to keep. All backups except the N newest ones will be deleted. You can also choose to delete arbitrary backup with call `DeleteBackup(id)`.

### Creating and verifying a backup

In RocksDB, we have implemented an easy way to backup your DB and verify correctness. Here is a simple example:
 
```cpp
    #include "rocksdb/db.h"
    #include "rocksdb/utilities/backupable_db.h"

    #include <vector>

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
        s = backup_engine->CreateNewBackup(db);
        assert(s.ok());
        std::vector<BackupInfo> backup_info;
        backup_engine->GetBackupInfo(&backup_info);
        s = backup_engine->VerifyBackup(1 /* ID */);  // if there exists more than one backup, get ID from backup_info
        assert(s.ok());
        delete db;
        delete backup_engine;
    }
```

This simple example will create a backup of your DB in "/tmp/rocksdb_backup".

Backups are normally incremental (see `BackupableDBOptions::share_table_files`). You can create a new backup with `CreateNewBackup()` and only the new data will be copied to backup directory (for more details on what gets copied, see [Under the hood](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F#under-the-hood)).

Once you have some backups saved, you can issue `GetBackupInfo()` call to get a list of all backups together with information on timestamp of the backup and the size (please note that sum of all backups' sizes is bigger than the actual size of the backup directory because some data is shared by multiple backups). Backups are identified by their always-increasing IDs.

When `VerifyBackups()` is called, it checks the file sizes in the backup directory against the sizes of the corresponding files in the db directory. However, we do not verify checksums since it would require reading all the data. Note that the only valid use case for `VerifyBackups()` is invoking it on a backup engine after that same engine was used for creating backup(s).

Checksums are always stored separately for any backed up file (including sst, log, and etc), and file size is embedded in filename when multiple databases share a single backup directory (see `Options::share_files_with_checksum`). These attributes uniquely identify files that can come from multiple RocksDB instances.

### Restoring a backup

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

`RestoreDBFromLatestBackup()` will restore the DB from the latest backup, i.e., the one with the highest ID. An alternative is `RestoreDBFromBackup()` which takes a backup ID and restores that particular backup. Checksum is calculated for any restored file and compared against the one stored during the backup time. If a checksum mismatch is detected, the restore process is aborted and `Status::Corruption` is returned.

### Advanced options

Let's say you want to backup your DB to HDFS. There is an option in `BackupableDBOptions` to set `backup_env`, which will be used for all file I/O related to backup dir (writes when backuping, reads when restoring). If you set it to HDFS Env, all the backups will be stored in HDFS.

`BackupableDBOptions::share_table_files` controls whether backups are done incrementally. If true, SST files will go under a "shared/" subdirectory. Conflicts can arise when different SST files use the same name (e.g., when multiple databases have the same target backup directory).

`BackupableDBOptions::share_files_with_checksum` controls how shared files are identified. If true, shared SST files are identified using checksum, size, and seqnum. This prevents the conflicts mentioned above when multiple databases use a common target backup directory.

`BackupableDBOptions::max_background_operations` controls the number of threads used for copying files during backup and restore. For distributed file systems like HDFS, it can be very beneficial to increase the copy parallelism.

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
5. Re-enable file deletions

### Further reading

For the implementation, see `utilities/backupable/backupable_db.cc`.
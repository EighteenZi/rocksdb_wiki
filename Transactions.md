RocksDB supports Transactions when using a TransactionDB or OptimisticTransactionDB.  Transactions have a simple BEGIN/COMMIT/ROLLBACK api and allow applications to modify their data concurrently while letting RocksDB handle the conflict checking.  RocksDB supports both pessimistic and optimistic concurrency control.  

Note that RocksDB provides Atomicity by default when writing multiple keys via WriteBatch. Transactions provide a way to guarantee that a batch of writes will only be written if there are no conflicts. Similar to a WriteBatch, no other threads can see the changes in a transaction until it has been written (committed).

### TransactionDB
When using a TransactionDB, all keys that are written are locked internally by RocksDB to perform conflict detection.  If a key cannot be locked, the operation will return an error.  When the transaction is committed it is guaranteed to succeed as long as the database is able to be written to.

A TransactionDB can be better for workloads with heavy concurrency compared to an OptimisticTransactionDB.  However, there is a small cost to using a TransactionDB due to the locking overhead.

	TransactionDB* txn_db;
	Status s = TransactionDB::Open(options, path, &txn_db);

	Transaction* txn = txn_db->BeginTransaction(write_options, txn_options);
	s = txn->Put(“key”, “value”);
	s = txn->Delete(“key2”);
	s = txn->Merge(“key3”, “value”);
	s = txn->Commit();
	delete txn;

### OptimisticTransactionDB
Optimistic Transactions provide light-weight optimistic concurrency control for workloads that do not expect high contention/interference between multiple transactions.

Optimistic Transactions do not take any locks when preparing writes. Instead, they rely on doing conflict-detection at commit time to validate that no other writers have modified the keys being written by the current transaction. If there is a conflict with another write (or it cannot be determined), the commit will return an error and no keys will be written.

Optimistic concurrency control is useful for many workloads that need to protect against occasional write conflicts. However, this many not be a good solution for workloads where write-conflicts occur frequently due to many transactions constantly attempting to update the same keys. For these workloads, using a TransactionDB may be a better fit. An OptimisticTransactionDB may be more performant than a TransactionDB for workloads that have many non-transactional writes and few transactions. 

	DB* db;
	OptimisticTransactionDB* txn_db;

	Status s = OptimisticTransactionDB::Open(options, path, &txn_db);
	db = txn_db->GetBaseDB();

	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options, txn_options);
	txn->Put(“key”, “value”);
	txn->Delete(“key2”);
	txn->Merge(“key3”, “value”);
	s = txn->Commit();
	delete txn;`


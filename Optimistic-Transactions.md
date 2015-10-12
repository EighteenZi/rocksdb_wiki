
Optimistic Transactions provide light-weight optimistic concurrency control for workloads that do not expect high contention/interference between multiple transactions.  

Optimistic Transactions do not take any locks when preparing writes.  Instead, they rely on doing conflict-detection at commit time to validate that no other writers have modified the keys being written by the current transaction.  If there is a conflict with another write (or it cannot be determined), the commit will return an error and no keys will be written.

Note that RocksDB provides Atomicity by default when writing multiple keys via WriteBatch.  Optimistic Transactions provide a way to guarantee that a batch of writes will only be written if there are no conflicts.  Similar to a WriteBatch, no other threads can see the changes in a transaction until it has been written (committed).

Optimistic concurrency control is useful for many workloads that need to protect against occasional write conflicts.  However, this many not be a good solution for workloads where write-conflicts occur frequently due to many transactions constantly attempting to update the same keys.  For these workloads, Pessimistic concurrency control is general a better solution.  (Pessimistic concurrency for RocksDB is currently under development.)

### Open an OptimisticTransactionDB
In order to use Optimistic Transactions, you need to Open an OptimisticTransactionDB similarly to the way you open a normal DB.

	DB* db;
  	OptimisticTransactionDB* txn_db;

  	Status s = OptimisticTransactionDB::Open(options, path, &txn_db);
  	db = txn_db->GetBaseDB();

### BEGIN and COMMIT a transaction
OptimisticTransactions have a basic Begin/Commit/Rollback API and support many of the same operations as a WriteBatch:

	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options, txn_options);
	txn->Put(“key”, “value”);
	txn->Delete(“key2”);
	txn->Merge(“key3”, “value”);
	Status s = txn->Commit();
	delete txn;

### Reading from a Transaction           
Optimistic Transactions also support easily reading the state of keys that are currently batched in a transaction:

	db->Put(write_options, “a”, “old”);
	db->Put(write_options, “b”, “old”);
	txn->Put(“a”, “new”);

	vector<string> values;
	vector<Status> results = txn->MultiGet(read_options, {“a”, “b”}, &values);
	//  The value returned for key “a” will be “new” since it was written by this transaction.
	//  The value returned for key “b” will be “old” since it is unchanged in this transaction.

### Setting a Snapshot

By default, Optimistic Transaction conflict checking validates that no one else has written a key *after* the time the key was first written in this transaction.  This isolation guarantee is sufficient for many use-cases.  However, you may want to guarantee that no else has written a key since the start of the transaction.  This can be accomplished by calling SetSnapshot() after creating the transaction.

Default behavior:

	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options);

	// Write to key1 OUTSIDE of the transaction
	db->Put(write_options, “key1”, “value0”);

	// Write to key1 IN transaction
	txn->Put(“key1”, “value1”);

	// Transaction will commit since the write to key1 outside of the transaction happened before it was written in this transaction.
	Status s = txn->Commit();
	assert(s.ok());
 
	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options);

	// Write to key1 IN transaction
	txn->Put(“key1”, “value1”);

	// Write to key1 OUTSIDE of the transaction
	db->Put(write_options, “key1”, “value0”);

	// Transaction will NOT commit since the write to key1 outside of the transaction happened AFTER it was written in this transaction.
	Status s = txn->Commit();
	assert(!s.ok());

Using SetSnapshot():

	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options);
	txn->SetSnapshot();

	// Write to key1 OUTSIDE of the transaction
	db->Put(write_options, “key1”, “value0”);

	// Write to key1 IN transaction
	txn->Put(“key1”, “value1”);

	// Transaction will NOT commit since key1 was written outside of this transaction after SetSnapshot() was called (even though this write
	// occurred before this key was written in this transaction).
	Status s = txn->Commit();
	assert(!s.ok());

### Repeatable Read
Similar to normal RocksDB DB reads, you can achieve repeatable reads when reading through a transaction by setting a Snapshot in the ReadOptions.

	read_options.snapshot = db->GetSnapshot();
	Status s = txn->GetForUpdate(read_options, “key1”, &value);
	…
	txn->Commit();
	db->ReleaseSnapshot(read_options.snapshot);

Note that Setting a snapshot in the ReadOptions only affects the version of the data that is read.  This does not have any affect on whether the transaction will be able to be committed.

If you have called SetSnapshot(), you can read using the same snapshot that was set in the transaction:
	read_options.snapshot = txn->GetSnapshot();
	Status s = txn->GetForUpdate(read_options, “key1”, &value);
	

### Guarding against Read-Write Conflicts:
GetForUpdate() will ensure that no other writer modifies any keys that were read by this transaction.

	// Start a transaction 
	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options);

	// Read key1 in this transaction
	Status s = txn->GetForUpdate(read_options, “key1”, &value);

	// Write to key1 OUTSIDE of the transaction
	db->Put(write_options, “key1”, “value0”);

	// Transaction will NOT commit since key1 was written outside of this transaction after it was read by this transaction.
	s = txn->Commit();
	assert(!s.ok());


	// Repeat the previous example but just do a Get() instead of a GetForUpdate()
	OptimisticTransaction* txn = txn_db->BeginTransaction(write_options);

	// Read key1 in this transaction
	Status s = txn->Get(read_options, “key1”, &value);

	// Write to key1 OUTSIDE of the transaction
	db->Put(write_options, “key1”, “value0”);

	// Transaction will commit since transactions only do conflict checking for keys read using GetForUpdate().
	s = txn->Commit();
	assert(s.ok());


### Tuning / Memory Usage

Internally, Optimistic Transactions need to keep track of which keys have been written recently.  The existing in-memory write buffers are re-used for this purpose.  Optimistic Transactions will still obey the existing **max_write_buffer_number** option when deciding how many write buffers to keep in memory.  In addition, using transactions will not affect flushes or compactions.

It is possible that switching to using an OptimisticTransactionDB will use more memory than was used previously.  If you have set a very large value for **max_write_buffer_number**, a typical RocksDB instance will could never come close to this maximum memory limit.  However, an OptimisticTransactionDB will try to use as many write buffers as allowed.  But this can be tuned by either reducing **max_write_buffer_number** or by setting **max_write_buffer_number_to_maintain** to a value smaller than max_write_buffer_number.

If **max_write_buffer_number_to_maintain** and **max_write_buffer_number** are too small, some transactions may fail to commit.  If this is the case, then the error message will suggest increasing **max_write_buffer_number_to_maintain**.

# Chapter 7 - Transactions

---

## What is A Transaction?

A transaction is a way to group several reads and writes together into a logical unit. Partially failed unit will be aborted as a whole and safely retried.

---

## Why Transaction?

Makes error handling much simpler for an application (no need to worry about partial failure).

---

## Safety Guarantees Provided by Transaction - ACID

### Safety Guarantees

- **A (Atomicity)**: A group of events either succeed or fail as a whole (no partial failure)
- **C (Consistency)**: All data tells the same story (e.g., bank accounts must always be balanced)
- **I (Isolation)**: Concurrently executing transactions are isolated from each other (as if executed serially)
- **D (Durability)**: Any data that's been written will not be forgotten (data has been written to nonvolatile storage or copied to certain number of nodes)

### Levels

- **On single-object** (writing a large JSON document can have partial failure): Can use log for crash recovery (atomicity) and a lock to guarantee isolation
- **On multi-object** (more crucial): Many distributed datastores have abandoned multi-object transactions because they are difficult to implement across partitions. However, there are many cases where writes to several objects need to be coordinated and the lack of transactions can make error handling very difficult. Transactions on multi-object level should not just be abandoned.

---

## Implementation

Most datastores with leaderless replication is not ACID compliant, and it's the application's responsibility to recover from errors.

---

## Weak Isolation Levels

In a perfect world, database guarantees **serializable isolation** (transactions have the same effect as if they ran serially). However, due to performance cost, it's common for systems to use weaker levels of isolation, which protects against some concurrency issues, but not all. Even ACID database use weak isolation, which introduces bugs hard to be detected.

### Race Conditions

- **Dirty reads**: See new data before it's been committed
- **Dirty writes**: Writes to data when another write hasn't been committed
- **Lost updates**: Two concurrent writes to increase a counter will result in one write missing
- **Read skew**: A write transaction happens and is committed in the middle of a read transaction, causing inconsistency in the values being read (e.g., balance transfer between two accounts)
- **Write skew**:
  - **Definition**: Write skew is a generalization of the lost update problem. Write skew can occur if two transactions read the same objects, and then update some of those objects (may be different objects). In the special case where they update the same object, you get a dirty write or lost update.
  - **Examples**:
    - Book meeting room conflicts
    - Multiplayer moving two figures to the same position
    - Two users claiming the same username at the same time
    - Double-spending
  - **Cause**: Phantoms (a write in one transaction changes the result of a search/condition-checking query in another transaction)

### Weak Isolation Levels

#### Read Committed

**Prevents**: Dirty read/writes

**Implementation**: Popular, default in **Oracle**, **PostgreSQL**, **SQLServer 2012**, **MemSQL**, etc.

**Details**: Use row-level locks on write; on read, save and return the old value before the new value is committed.

**Pros & Cons:**
- ✅ No dirty reads (only see data has been committed), no dirty writes (only overwrite data has been committed)
- ❌ Does not prevent the race condition between two counter increments

#### Snapshot Isolation and Repeatable Read

**Prevents**: Read skew - if the data is changed by another transaction during the read transaction, the read transaction sees only the old data throughout.

**How**: Reads block other reads; writes block other writes. (Readers never block writers, writers never block readers)

**Implementation**: Supported by **Oracle**, **PostgreSQL**, **MySQL**, **SQLServer**, etc. Useful especially for read-only transactions.

**Pros & Cons:**
- ✅ No read skew (each transaction reads from a consistent snapshot of the database), good for backups and analytics
- ✅ Reads can be processed without blocking concurrent writes

#### Preventing Lost Updates

**Classic scenario**: Two concurrent writes to increase a counter will result in one write missing.

**Solutions:**

1. **Atomic write operations** (application code read and write in one single operation):
   ```sql
   Update counters SET value = value + 1 WHERE key = 'foo'
   ```

2. **Explicit locking**: Application code locks objects that are going to be updated
   ```sql
   BEGIN TRANSACTION;
   SELECT * FROM figures WHERE name = 'robot' AND game_id = 222
   FOR UPDATE; -- indicates database takes a lock on all rows returned by this query
   -- can add some validation logic here
   UPDATE figures SET position = 'c4' WHERE id = 1234;
   COMMIT;
   ```

3. **Automatically detecting lost updates**: Allow parallel execution of transactions, abort and retry if the transaction manager detects a lost update. This is implemented on the database side and application code no longer need to use any special database features as they do with the previous 2 solutions.

4. **Compare-and-set**: Often found in databases that don't provide transactions; only allow update to happen when it is the same value as you last read it.
   ```sql
   UPDATE wiki_page SET content = 'new content'
   WHERE id = 1234 AND content = 'old content' -- only updated when content matched the old content
   ```

5. **Conflict resolution and replication** (for replicated databases): Allow concurrent writes to create several conflicting versions of a value (siblings), and to use application code to resolve and merge these versions.

#### Preventing Write Skew (a generalized type of lost updates)

- **Automatically preventing write skew** - requires true serializable isolation (difficult)
- **Constraints that involves multiple objects** (most DB don't have built-in support, need to implement with triggers or materialized views)
- **Explicitly lock the rows the transaction depends on** (if serializable isolation level can't be used):
  ```sql
  BEGIN TRANSACTION;
  SELECT * FROM doctors
      WHERE oncall = true
      AND shift_id = 1234 FOR UPDATE; -- lock all rows returned by this query
  UPDATE doctors
      SET oncall = false
      WHERE name = 'Alice'
      AND shift_id = 1234;
  COMMIT;
  ```
  > **Note**: When the condition checking verifies if a row exists (e.g., meeting room booking), no lock can be added to the non-existing row. Then this solution doesn't work under this scenario. Materializing conflicts is needed.

- **Materializing conflicts** (last resort): When locking object isn't possible (objects doesn't exist), can introduce a new table which keeps track of conflicts and locks. (error-prone). A serializable isolation level is much preferable.

---

## Serializable Isolation

### Definition

Even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without any concurrency.

### Why?

- **Stronger isolation guarantee**: Protects all race conditions
- **Using weaker isolation levels is error-prone**:
  - Isolation levels are hard to understand, and have inconsistent implementations in different databases
  - For large applications, it's difficult to tell whether it's safe to run at a particular isolation level
  - Race conditions are nondeterministic and detecting them is hard

### Techniques (for single-node databases)

#### 1. Actual Serial Execution

**Definition**: Executing only one transaction at a time, in serial order, on a single thread.

**Implementation**: **Redis**

**Actual Serial Execution through stored procedures:**

- **Problem**: Multi-statement transactions reduces throughput because the database spends too much time waiting for the application to send the next query for the current transaction.
- **Solution**: Systems with single-threaded serial transaction processing don't allow multi-statement transactions. Instead, the application must submit the entire transaction code to the database ahead of time, as a **stored procedure**.

**Pros & Cons:**
- ✅ Makes concurrency control much simpler
- ❌ Limits throughput (especially when write is heavy): can partition the data, each partition can have its own thread (transaction throughput can scale linearly with the number of CPU cores, but introduces significant overhead when cross-partition write is needed, e.g., secondary index)
- ❌ Every transaction must be small and fast (otherwise will stall all other transactions)

#### 2. Two-Phase Locking (2PL, widely used)

**Idea**: Writes don't just block other writes, they also block reads and vice versa (stronger than snapshot isolation where readers don't block writes and vice versa).

**Implementation**: Used by the serializable isolation level in **MySQL** and **SQL Server**.

**Steps:**

- A read must acquire lock in shared mode. If an exclusive lock already exists, must wait.
- A write must acquire an exclusive lock. If a lock already exists, must wait.
- A transaction with read/write combined need to update the shared lock to exclusive lock when it writes
- A transaction must hold the lock until the end of the transaction

> **Note**: Deadlock - database automatically detects deadlocks between transactions and aborts one of them. The aborted transaction needs to be retried by the application.

**Disadvantage - Performance**: Significantly worse under two-phase locking than under weak isolation (due to reduced concurrency); unstable latencies (just one slow transaction cause the rest transactions to halt); deadlocks can be frequent.

**Types of locks:**

- **Predicate locks**:
  - **Goal**: Locking only the objects being read/written without blocking the other objects
  - **How**: Rather than belonging to a particular object, it belongs to all objects that match some search conditions (e.g., lock bookings of room 123 between 1-2pm)
  - **Pros & Cons:**
    - ✅ Applies even to objects that do not yet exist in the database, but which might be added in the future (phantoms)
    - ❌ If there are many locks by active transactions, checking for matching locks becomes time-consuming. (use index-range locks)

- **Index-range locks**:
  - **Idea**: It's safe to simplify a predicate by making it match a greater set of objects (e.g., lock bookings of room 123 at any time, or lock bookings of all rooms between 1-2pm; can use an index to lock or lock the whole table if no suitable index exist)
  - **Pros & Cons:**
    - ❌ Less precise than predicate locks
    - ✅ Lower overheads (worth trade-off)

#### 3. Serializable Snapshot Isolation

**Highlight**: Serializable isolation with good performance (avoiding unnecessary aborts); used both in single-node and distributed databases; still young compared to other concurrency control mechanisms, has the possibility of being fast enough to become the new default in the future.

**Optimistic concurrency control**: Allows concurrent transactions, check race conditions when transaction is committed and see if it's necessary to abort until then. (2PL locking uses pessimistic concurrency control where it assumes all concurrency transactions would result in race conditions).

**How does it detect phantoms** (change of read query results from a concurrent write)?

- **Detecting stale MVCC reads**: The database tracks when a transaction ignores another transaction's writes when they are not committed. When the transaction wants to commit, the database checks whether any of the ignored writes have now been committed. If so, the transaction must be aborted.
- **Detecting writes that affect prior reads**: The database tracks which transactions are reading the data. When a transaction writes to the database, it must look in the indexes for any other transactions that have read the affected data. If yes, abort.

**Advantage - Performance**: Makes query latency much more predictable and less variable (no need to wait for locks held by another transaction) at the cost of extra bookkeeping of concurrent transactions' behaviors; not limited to the throughput of a single CPU core.

---

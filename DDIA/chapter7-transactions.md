# Chapter 7 - Transactions
## What is A Transaction?
A transaction is a way to group several reads and writes together into a logical unit. Partially failed unit will be aborted as a whole and safely retried.
## Why Transaction?
Makes error handling much simpler for an application (no need to worry about partial failure)
## Safety Guarantees Provided by Transction - ACID
* Safety Guarantees
    * A (atomicity): a group of events either succeed or fail as a whole (no partial failure)
    * C (consistency): all data tells the same story (eg. bank accounts must always be balanced)
    * I (isolation): concurrently executing transactions are isolated from each other (as if executed serially)
    * D (durability): any data that's been written will not be forgotten (data has been written to nonvolatile storage or copied of certain number of nodes) 
* Levels
    * on single-object (writing a large JSON document can have partial failure): can use log for crash recovery (atomicity) and a lock to guarantee isolation
    * on multi-object (more crucial): many distributed datastores have abandoned multi-object transactions because they are difficult to implement across parttitions. However, there are many cases where writes to several objects need to be corrdinated and the lack of transactions can make error handling very difficult. Transactions on multi-object level should not just be abandoned. 
## Implementation
Most datastores with leaderleess replication is not ACID compliant, and it's the application's responsibility to reecover from errors.
## Weak Isolation Levels
In a perfect world, database guarantees **serializable isolation** (transactions have the same effect as if they ran serially). However, due to performance cost, it's common for systems to use weaker levels of isolation, which protects against some concurrency issues, but not all. Even ACID database use weak isolation, which introduces bugs hard to be detected.
* Race Conditions
    * dirty reads: see new data before it's been committed
    * dirty writes: writes to data when another write hasn't been committed
    * lost updates: two concurrent writes to increase a counter will result in one write missing
    * read skew: a write transaction happens and is committed in the middle of a read transaction, causing inconsistency in the values being read. (eg. balance transfer b/t two accounts)
    * write skew:
        * definition: write skew is a generalization of the lost update problem. Write skew can occur if two transactions read the same objects, and then update some of those objects (may be different objects). In the special case where they update the same object, you get a dirty write or lost update. 
        * examples:
            * book meeting room conflicts
            * multiplayer moving two figures to the same position
            * two user claiming the same username at the same time
            * double-spending
        * cause: phantoms (a write in one trasaction changes the result of a search/condition-checking query in another transaction). 
* Weak Isolation Levels
    * Read Committed: 
        * prevents dirty read/writes
        * implementation: popular, default in Oracle, PostgreSQL, SQLServer 2012, MemSQL, etc.
            * details: use row-level locks on write; on read, save and return the old value before the new value is committed.
        * (+) no dirty reads (only see data has been committed), no dirty writes (only overwrite data has been committed). (-) does not prevent the race condition between two counter increments
    * Snapshot Isolation and Repeatable Read
        * prevents read skew: if the data is changed by another transaction during the read transaction, the read transaction seed=s only the old data throughout.
        * how: reads block other reads; writes block other writes. (readers never block writers, weriters never block readers)
        * implementation: supported by Oracle, PostgreSQL, MySQL, SQLServer, etc. Useful especially for read-only transactions
        * pros & cons
            (+) no read skew (each transaction reads from a consistent snapshot of the database), good for backups and analytics
            (+) reads can be processed without blocking concurrent writes.
    * Preventing Lost Updates
        * classic scenario: two concurrent writes to increase a counter will result in one write missing
        * solutions:
            * atomic write operations: (application code read and write in one single operation)
                ```
                Update counters SET value = value + 1 WHERE key = 'foo'
                ```
            * explicit locking: 
            application code locks objects that are going to be updated

                ```
                BEGIN TRANSACTION;
                SELECT * FROM figures WHERE name = 'robot' AND game_id = 222
                FOR UPDATE; -- indicates database takes a lock on all rows returned by this query
                -- can add some validation logic here
                UPDATE figures SET position = 'c4' WHERE id = 1234;
                COMMIT;
                ```
            * Automatically detecting lost updates
            allow parallel execution of transactions, abort and retry if the transaction manager detects a lost update. This is implemented on the database side and application code no longer need to use any special database features as they do with the previous 2 solutions
            * Compare-and-set
            often found in database that don't provide transactions; only allow update to happen when the it is the same value as you last read it. 
                ```
                UPDATE wiki_page SET cotent = 'new content'
                WHERE id = 1234 AND cotent = 'old content' -- only updated when content matched the old content
                ```
            * conflict resolution and replication (for replicated databases)
            allow concurrent writes to create several conflicting versions of a value (siblings), and to use application code to resolve and merge these versions.
    * Preventing Write Skew (a generalized type of lost updates)
        * automatically preventing write skew - requires true serializable isolation (difficult)
        * constraints that involves multiple objects (most db don't have built-in support, need to implement with triggers or materialized views)
        * explicitly lock the rows the transaction depends on (if serializable isolation level can't be used)
            ```
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
            note: when the condition checking verifies if a row exists (eg. meeting room booking), no lock can be added to the non-existing row. Then this solution doesn't work under this scenarion. Materializing conflicts is needed.
        * materializing conflicts (last resort): when locking object isn't possible (objects doesn't exist), can introduce a new table which keeps track of conflicts and locks. (error-prone). A serializable isolation level is much preferable.
            
## Serializable Isolation
### Definition:
Even through transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without any concurrency.
### Why?
* Stronger isolation guarantee: protects all race conditions 
* Using weaker isolation levels is error-prone:
    * Isolaiton levels are hard to understand, and have inconsistent implementations in different databases
    * For large applications, it's difficult to tell whether it's saft to run at a particular isolation level
    * Race conditions are nondeterministic and detecting them is hard.
### Techniques (for single-node databases)
1. Actual Serial Execution
    * definition: executing only one transaction at a time, in serial order, on a single thread
    * implementation: Redis
    * actual Serial Execution through stored procedures:
        * problem: multi-statement transactions reduces throughput because the database spends too much time waiting for the application to send the next query for the current transaction.
        * solution: systems with single-threaded serial transaction processing don't allow multi-statement transactions. Instead, the application must submit the entire trasaction code to the database ahead of time, as a **stored procedure**
    * pros & cons:
        * (+) makes concurrency control much simpler
        * (-) limits throughput (especially when write is heavy): can partition the data, each partition can have its own thread (transaction throughput can scale linearly with the number of CPU cores, but introduces significant overhead when cross-partition write is needed, eg. secondary index)
        * (-) every transaction must be small and fast (otherwise will stall all other transactions)
2. Two-Phase Locking (2PL, widely used)
    * idea: writes don't just block other writes, they also block reads and vice versa (stronger than snapshot isolation where readers don't block writes and vice versa)
    * implementation: used by the serializable isolation level in MySQL and SQL Server.
    * steps:
        * a read must acquire lock in shared mode. If an exclusive lock already exists, must wait.
        * a write must acquire an exclusive lock. If a lock already exists, must wait.
        * a transaction with read/write combined need to update the shared lock to exclusive lock when it writes
        * a transaction must hold the lock until the end of the transaction

        deadlock: database automatically detects deadlocks between transactions and aborts one of them. The aborted transaction needs to be retried by the application.
    * disadvantage - performance: significantly worse under two-phase locking than under weak isolation (due to reduced concurrency); unstable latencies (just one slow transaction cause the rest transactions to halt); deadlocks can be frequent
    * Types of locks
        * predicate locks: 
            * goal: locking only the objects being read/written without blocking the other objects
            * how: rather than belonging to a particular object, it belongs to all objects that match some search conditions (eg. lock bookings of room 123 between 1-2pm)
            * pros & cons:
                (+) applies even to objects that do not yet exist in the database, but which might be added in the future (phantoms)
                (-) if there are many locks by active transactions, checking for matching locks becomes time-consuming. (use index-range locks)
        * index-range locks
            * idea: it's safe to simplify a predicate by making it match a greater set of objects (eg. lock bookings of room 123 at any time, or lock bookings of all rooms between 1-2pm; can use an index to lock or lock the whole table if no suitable index exist)
            * pros & cons:
                (-) less precise than predicate locks
                (+) lower overheads (worth trade-off)
3. Serializable Snapshot Isolation
* highlight: serailizable isolation with good performance (avoiding unnecessary aborts); used both in single-node and distributed databses; still young compared to other concurrency control mechanisms, has the possibility of being fast enough to become the new default in the future
* optimistic concurrency control: allows concurrent transactions, check race conditions when transaction is committed and see if it's necessary to abort until then. (2PL locking uses pessimistic concurrency control where it assumes all concurrency transactions would result in race conditions). 
* how does it detect phantoms (change of read query results from a concurrent write)?
    * detecting stale MVCC reads: the database tracks when a transaction ignores another transaction's writes when they are not committeed. When the transaction wants to commit, the database checks whetheer any of the ignored writes have now been committed. If so, the transaction must be aborted. 
    * detecting writes that affect prior reads: the database tracks which transactions are reading the data. When a transaction writes to the database, it must look in the indexes for any other transactins that have read the affected data. If yes, abort.
* advantage - performance: makes query latency much more predictable and less variable (no need to wait for locks held by another transaction) at the cost of extra bookkeeping of concurrent transactions' behaviors; not limited to the throughput of a single CPU core.
    

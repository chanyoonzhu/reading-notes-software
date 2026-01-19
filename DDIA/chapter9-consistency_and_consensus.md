# Chapter 9 - Consistency and Consensus

This chapter goes through some examples of algorithms and protocols for building fault-tolerant distributed systems.

---

## How to build a fault-tolerant system?

Find some general-purpose abstractions with useful guarantees, implement them once, and then let applications rely on those guarantees (e.g., using transaction implemented in DB) so that they can ignore some of the problems with distributed systems.

---

## What are the most important abstractions for distributed systems?

**Consensus** (getting all of the nodes to agree on something).

---

## Strong Consistency Guarantee - Linearizability

### Overview

- **Linearizability** is the strongest consistency guarantee: a linearizable register (object) behaves as if there's only a single database, and every operation appears to take effect atomically at one point in time.
- **Problem it solves**: Eventual consistency is a very weak guarantee (doesn't guarantee when the replicas will converge). Stronger consistency models need to be explored.
  - **Pros**: Easier to use from applications perspective
  - **Cons**: Worse performance, less fault-tolerant
- A useful guarantee, but few systems are actually linearizable due to its cost on availability and the fact that linearizability is slow (especially with large network delays). Weaker consistency guarantees (e.g., ordering guarantees) is much faster.
- **Linearizable**: All requests and responses can be arranged into a valid sequential (total) order (p.328 figure 9-4).

### Linearizability vs. Serializability

Both mean something like "can be arranged in a sequential order".

- **Serializability**: An isolation property of transactions: even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without any concurrency.
- **Linearizability**: A recency guarantee on reads and writes of a **single object** (doesn't assume any transaction isolation).

> **Note**: A database may provide both serializability and linearizability (aka. **strict serializability**). Of the three techniques of **serializable isolation**, actual serial execution and two-phase locking provide strict serializability, while serializable snapshot isolation does not.

### Linearizability Use Cases

- **Locking and leader election**: Single-leader replication system needs to ensure that there is indeed only one leader (no brain split); coordination services like **Apache ZooKeeper** are often used to implement distributed locks and leader election.
- **Constraints and uniqueness guarantees**: (e.g., unique user ID, bank account balance not negative)
- **Cross-channel timing dependencies**: Communication went faster through one channel compared to the other, which is supposed to arrive first (e.g., a webserver stores file and send message to resize the file through two different channels. However, the message queue received the resizing message first, when file hasn't been stored into storage yet)

### Implementing Linearizable Systems

**What replication methods can be made linearizable?**

1. **Single-leader replication**: Potentially linearizable if you make reads from the leader, or from synchronously updated followers; violates linearizability if a false leader (split brain) serves requests
2. **Consensus algorithm**: Linearizable (has resemblance to single-leader replication with no split brain)
3. **Multi-leader replication**: Not linearizable (can produce conflict writes)
4. **Leaderless replication**: Probably not linearizable ("Last write wins" based on time-of-day clocks with possible clock skews; sloppy quorums)

**Linearizability and quorums**: It seems strict quorum reads and writes should be linearizable. However, when we have variable network delays (replication lag), it is possible to have race conditions. Therefore, it's safest to assume that a leaderless system with Dynamo-style replication does not provide linearizability.

### The Cost of Linearizability

Applications that don't require linearizability can be more tolerant of network problems. (the CAP theorem)

- **The CAP theorem**: Consistency (linearizability), Availability, pick 1 out of 2 when partitioned.

### Sequential Consistency (Timeline Consistency)

Slightly weaker than linearizability. It only guarantees linearizable writes, not linearizable reads (good enough for assigning unique username).

---

## A Weaker Consistency Guarantee - Ordering Guarantees

### Overview

- Only guarantees causality, i.e., **causally consistent**; two events are ordered if they are causally related, but they are incomparable if they are concurrent.
- A weaker consistency than linearizability: linearizability guarantees total ordering (including events without causal relationships); in many cases, systems that appear to require linearizability only really require causal consistency.
  - **Advantage**: The strongest possible consistency model that does not slow down due to network delays (like linearizability does), and remains available in the face of network failures.
- **Current status**: Promising direction, not yet made into production.

### Pros & Cons

- ✅ Does not have the coordination overhead of linearizability
- ✅ Much less sensitive to network problems
- ❌ Some things like unique username assignment cannot be implemented (does not know whether another node is concurrently in the process of registering the same name). (need to achieve **consensus**)

### Capturing Causal Dependencies

**How?** - When a replica processes an operation, it must ensure that all causally preceding operations have already been processed; if some preceding operation is missing, the later operation must wait until the preceding operation has been processed. It tracks causal dependencies across the entire database (not just for a single key) using **version vectors**. The version number from the prior operation is passed back to the database on a write in order to determine the causal ordering. (p.189)

**Methods:**

1. **Sequence number ordering**: Using sequence numbers or timestamps (something monotonically increasing) to order events that is consistent with causality. **Lamport timestamps** is usually used.
   - **Lamport timestamps** generate sequence numbers across nodes: each node has a unique ID, and each node keeps a counter of the number of operations it has processed (counter, node ID). If you have two timestamps, the one with a greater counter value is the greater timestamp; if the counter values are the same, the one with the greater node ID is the greater timestamp. When a node sees a counter value greater than its own counter value (contained in a client request/response), it immediately increases its own counter to that maximum.
   - **Disadvantage**: Not enough for implementing creating unique usernames (the total order of operations only emerges after the operation is completed; we need to know when that order is finalized - need total order broadcast).

2. **Total order broadcast**:
   - **Definition**: (delivered exactly once, in the same order)
     - **Reliable delivery**: No messages are lost; if a message is delivered to one node, it is delivered to all nodes
     - **Totally ordered delivery**: Messages are delivered to every node in the same order
   - **Implementation**: **ZooKeeper**
   - **Used for:**
     - **Database replication**: If every message represents a write to the database, and every replica processes the same writes in the same order, then the replicas will remain consistent with each other (aka. **state machine replication**)
     - **Implementing serializable transactions**
     - **Implementing lock service** that provides fencing tokens
   - **Implementing linearizable storage using total order broadcast**: If you have total order broadcast, you can build linearizable storage on top of it (e.g., unique usernames).
     - **How?**
       1. Append a message to the log indicating the username you're claiming
       2. Read the log, and wait for the message you appended to be delivered back to you
       3. Check for any messages claiming the username that you want. If the first message is your own, then the claiming is successful, otherwise abort. (this is based on the total ordering of messages)
   - **Implementing total order broadcast using linearizable storage**:
     - **How?**
       1. For a message to be sent through total order broadcast, you increment-and-get the linearizable integer, and then attach the value as a sequence number to the message.
       2. Send the message to all nodes (resending lost messages), and the recipients will deliver the messages consecutively by sequence number (if a number in the middle is missing, the node will wait)
     - **Hard to have a linearizable storage when**: Network connections to the node keeping the linearizable integer are interrupted; restoring the value when that node fails.

---

## Distributed Transactions and Consensus

### Overview

- **The most important problem in distributed system**: Get several nodes to agree on something irrevocably.
- **Definition of consensus problem**: One or more nodes may propose values, and the consensus algorithm decides on one of those values.

### Equivalent Problems

That can be reduced to consensus problems:

1. **Linearizable compare-and-set registers**: The register (object/variable) needs to atomically decide whether to set its value, based on whether its current value equals the parameter given in the operation
2. **Atomic transaction commit**: A database must decide whether to commit or abort a distributed transaction
3. **Total order broadcast**: The messaging system must decide on the order in which to deliver messages
4. **Locks and leases**: When several clients are racing to grab a lock or lease, the lock decides which one successfully acquire it
5. **Membership/coordination service**: Given a failure detector (e.g., timeouts), the system must decide which nodes are alive, and which should be considered dead because their sessions timed out
6. **Uniqueness constraint**: When several transactions concurrently try to create conflicting records with the same key, the constraint must decide which one to allow and which should fail with a constraint violation

### Consensus Algorithm Properties

A consensus algorithm must satisfy the following properties:

1. **Uniform agreement**: No two nodes decide differently
2. **Integrity**: No node decides twice
3. **Validity**: If a node decides value `v`, then `v` was proposed by some node
4. **Termination**: Every node that does not crash eventually decides some value

**Pros & Cons:**
- ✅ Consensus algorithms are a huge breakthrough for distributed systems. They can implement linearizable atomic operations in a fault-tolerant way.
- ❌ Most consensus algorithms assume a fixed set of nodes participating in voting; blocked on synchronous replication when voting on proposals; sensitive to network problem (uses timeouts to detect node failure)

### Use Cases

- **Leader election**: No split brain
- **Atomic commit**: A transaction either commit or rolled back on all nodes

### Algorithms

#### 1. Two-Phase Commit (2PC)

- **Overview**: Most common way of solving atomic commit in a distributed system; implemented in various databases, messaging systems, and application servers. Not a very good consensus algorithm - weak fault-tolerance when coordinator fails.
- **When should a node in a distributed system commit?** Only when it's certain that all other nodes in the transaction are also going to commit.
- **Implementation**: Most "NoSQL" distributed datastores do not support distributed atomic transactions, but various clustered relational systems do.

**Steps:**

1. Reading and writing data on multiple database nodes as normal
2. The coordinator (in the application or a dedicated service) sends a prepare request to each of the participating nodes, asking them whether they're able to commit (phase 1)
3. If all participants answer "yes", the coordinator sends out a commit request to nodes to commit, otherwise sends an abort request. (phase 2)

**Details**: Each read/write sent to nodes are tracked with a globally unique transaction ID (managed by coordinator) if abort is needed. Once a node answers "yes", it must commit the transaction in phase 2 if the coordinator chooses to do so (even if node crashes, it will commit after it comes back). Once a coordinator decides all nodes should commit, all nodes must commit in spite of request fail or timeout (the coordinator keeps retrying until successful). If the coordinator crashes, the participants can do nothing but wait. The coordinator must write its commit or abort decision to a transaction log on disk before sending out final requests to help recover from crash.

**Pros & Cons:**
- ✅ Result is similar as a regular single-node atomic commit
- ❌ Does not satisfy the "Termination" properties of a consensus algorithm: a blocking commit when the coordinator crashes and all nodes waiting for it to recover at phase 2. (performance penalty - database cannot release those locks until the transaction commits or aborts and no other transaction can modify those rows until then; many cloud services choose not to implement distributed transactions due to this reason)
- ❌ Single point of failure if the coordinator runs only on a single node
- ❌ Coordinator log makes the application not stateless (need to store the transaction log in disk, like a database)

**XA Transactions**: A standard for implementation across heterogeneous technologies (participants of 2PC use two or more different technologies).

- A C API for interfacing with a transaction coordinator.
- The coordinator in application calls the XA API to find out whether an operation should be part of a distributed transaction, and ask the participants to prepare, commit, or abort.

#### 2. Fault-Tolerant Consensus Algorithms

- **The best known fault-tolerant consensus algorithms are**: **Viewstamped Replication**, **Paxos**, **Raft**, and **Zab**
- Are in fact **total ordering broadcast** (which is equivalent to repeated rounds of consensus)
- Elects a "leader" in each epoch using an incremented epoch number to track votes. The leader is voted from a quorum of nodes
- **Biggest difference from 2PC**: Consensus is arrived from a majority of nodes (quorum) whereas in 2PC, consensus need to be obtained from all nodes

### ZooKeeper - A Service Implementing Consensus Algorithm

- **What is ZooKeeper used for**: Users "outsource" some of the work of coordinating nodes to ZooKeeper.

**Use cases:**

1. Choose a leader (for databases, job schedulers, etc.)
2. Deciding which partition to assign to which node
3. Service discovery (actually not requires consensus that much, DNS can do the job)
4. Membership services: Determines which nodes are currently active and live members of a cluster

---

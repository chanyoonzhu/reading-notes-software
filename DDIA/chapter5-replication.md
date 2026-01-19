# Chapter 5 - Replication

---

## Why Replication?

Replication keeps data consistent between nodes. Key benefits include:

- **Latency**: Keep data geographically closer to the user
- **Availability**: Can read from multiple nodes when one fails
- **Throughput**: Spread read load across multiple machines

---

## Replication Algorithms

### Single-Leader Replication
- One leader and many followers
- All writes are sent to the leader only
- All nodes can serve read queries (built-in feature for many relational databases)
- **Challenge**: Failover (handling leader failure) can be troublesome

### Multi-Leader Replication
- More than one node accepts writes
- **Challenge**: Hard to resolve write conflicts, should be avoided

### Leaderless Replication
- No leader
- Read requests sent to multiple nodes to resolve conflict values
- No need for failover when quorum (number of read/write confirmations) is properly configured

---

## Replication Configuration

### Synchronous vs. Asynchronous

- **Full Synchronous**: Can block writes, so is impractical
- **Asynchronous**: Widely used, but data among different nodes can be inconsistent

---

## Consistency Guarantee

### 1. Weakest Guarantee: Eventual Consistency

### 2. Stronger Guarantees

- **Read-after-write consistency**: A single user is guaranteed to see the change after writes from herself
- **Monotonic reads**: User will not see out-of-sequence data
- **Consistent prefix reads**: Writes causally related to each other show in the order in which the writes occur

> **Note**: Stronger guarantees can be made simpler (for the application layer) using **transactions** at the database layer.

### 3. Strongest: Strong Consistency

> **Note**: Impractical in most distributed systems.

---

## Replication Types

### Single-Leader Replication

#### How is Data Replicated?

Write logs propagate from leader to followers. **Logical (row-based) log** (p.160) is the most common.

#### Node Outages

**Followers:**
- Catch up changes using the log of the leader and itself

**Leader:**
- Consuming from the new leader (aka. **failover**) with steps:
  1. Figuring out leader has failed (with timeouts)
  2. Choosing a new leader (with most up-to-date data)
  3. Using the new leader

**Failover Challenges:**
- **Durability issue**: New leader not caught up with old leader
- **Inconsistency/reuse of primary keys**
- **Split brain**: Two leaders
- **Timeout configuration**

> **Note**: As a result, some teams still prefer performing failover manually.

---

### Multi-Leader Replication

#### How?

A leader in each datacenter.

#### Implementation

- **CouchDB**

#### Current Status

Often implemented with external tools.

#### Pros & Cons

**Pros:**
- Reduce latency (can write to more than one node)
- Better tolerance of datacenter outages:
  - No failover needed
  - Other datacenters still work
  - Can wait for the failed datacenter to come back
- Better tolerance of network issues

**Cons:**
- Write conflicts (autoincrementing keys, triggers, integrity constraints)

#### Topologies

- **Circular**
- **Star**
- **All-to-all** (common, better fault-tolerance)
  - One node failure does not impact other nodes like circular and star topology do

> **Caveat**: Handling infinite loops.

---

### Leaderless Replication

#### How it Works?

- Reads/Writes sent to multiple nodes
- Need `w` write confirmations and `r` read confirmations to guarantee read/write success, otherwise returns an error
- Replication happens asynchronously and without enforcing the order of writes

#### Pros & Cons

**Pros:**
- Tolerate conflicting concurrent writes, network interruptions, and latency spikes

#### Implementation

- **DynamoDB**
- **Riak**
- **Cassandra**
- **Voldemort**

#### Data Catch Up

- **Read repair**: Lazy repair on read
- **Anti-entropy process**: Background process

#### Read/Write Quorums

As long as `w` (write confirmed nodes) + `r` (read queried nodes) > `n` (number of nodes), we expect to get an up-to-date value when reading.

> **Principle**: At least one of the `n` replicas you read stores the most recent successful write - "pigeon hole" principle. In other words, `r` and `w` are the minimum number of votes required for the read or write to be valid.

**Configuration:**

- **`w`, `r` balancing**:
  - For write heavy: decrease the value of `w` and increase the value of `r`
  - For read heavy: vice versa
- Can pick `w`, `r` that `w + r < n` to achieve lower latency and higher availability (may see stale value)
- **Multi-datacenter**:
  - Can configure `n` to be the number of nodes in all datacenters (Cassandra)
  - Or the number of local nodes (Riak)

**Limitations:**
- Can have edge cases where still see stale value

**Sloppy Quorum:**
- Writes and reads require `w` and `r` successful responses by not necessarily from the designated `n` "home" nodes (e.g., nodes in other datacenter)
- Whether sloppy quorum is used can usually be configured in DB implementations

---

## Handling Write Conflicts

### What is Concurrency?

Neither event A and B happens before each other or knows about each other.

### How Should Concurrency be Handled?

- When events are **not concurrent**: The later operation should overwrite the earlier
- When events are **concurrent**: We need to resolve the conflict

### Asynchronous Conflict Detection

To allow writes to multi nodes, asynchronous conflict detection preferred over synchronous (otherwise better choose single-leader replication).

### Approaches

#### 1. Avoid Conflict

**(For multi-leader replication)** All writes for a particular record go through the same leader (recommended).

#### 2. Converging to a Consistent State

##### Last Write Wins (LWW)

- Using timestamp, etc.
- Default implementation in many replicated databases

**Pros & Cons:**
- ❌ **Cons**: Lost updates

##### Other Approaches

- **Replica with higher ID wins**
- **Merge all values**
- **Record all values, use application code (or prompt user) to choose value** (common)

**Steps:**

1. **Capturing conflicts (siblings)** (p.187):
   - Using version number
   - Merge when override
   - Append when concurrent

2. **Handling conflicts aka. merging siblings** (concurrent values):
   - Union
   - Add deletion marker (tombstone) to removed item
   - Merging siblings in application code is error-prone
   - Automatic conflict resolution is still under research

3. **Write conflicts with multi replica: Version vectors**:
   - A version number per replica
   - Each replica increments its own version number
   - Also keeps track of the version numbers it has seen from each of the other replicas
   - This indicates which values to overwrite and which to keep as siblings

**Types:**
- Resolve conflict on write
- Resolve conflict on read

#### 3. Automatic Conflict Resolution

> **Status**: Early stage, promising.

---

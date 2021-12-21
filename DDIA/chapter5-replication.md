# Chapter 5 - Replication
## Why Replication?
* latency: keep data geographically closer to the user
* availability: can read from multiple nodes when one fails
* throughput: spread read on multiple machines
## Replication Algorithms
* single-leader: one leader and many followers; all writes are sent to the leader only; all nodes can serve read queries (built-in feature for many relational databases)
* multi-leader
* leaderless
## Replication Configuration
* Synchronous vs. asynchronous: full synchronous can block writes so is impractical; asynchronous is widely used but data among different nodes can be inconsistent.
## Consistency Guarantee
1. weakest guarantee: eventual consistency
2. stronger guarantees:
    * read-after-write consistency: a single user is guaranteed to see the change after writes from herself
    * monotonic reads: user will not see out-of-sequence data
    * consistent prefix reads: writes causally related to each other show in the order in which the writes occur
3. strongest: strong consistency (impractical)

Stronger guarantees can be made simpler (for the application layer) using **transactions** at the database layer

## Single-leader Replication
### How is Data Replicated?
Write logs propagate from leader to followers. Logical (row-based) log (p.160) is the most common.
### Node Outages
* followers: catch up changes using the log of the leader and itself
* leader: consuming from the new leader (aka. failover) with steps 1. figuring out leader has failed (with timeouts) 2. choosing a new leader (with most up-to-date data) 3. using the new leader
    * failover can go wrong in many ways: durability issue - new leader not caught up with old leader; inconsistency/reuse of primary keys; split brain(two leaders); timeout configuration. As a result, some teams still prefer performing failover manually.
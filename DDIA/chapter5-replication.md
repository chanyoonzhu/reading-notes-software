# Chapter 5 - Replication
## Why Replication?
Keeps data consistent between nodes
* latency: keep data geographically closer to the user
* availability: can read from multiple nodes when one fails
* throughput: spread read on multiple machines
## Replication Algorithms
* single-leader: one leader and many followers; all writes are sent to the leader only; all nodes can serve read queries (built-in feature for many relational databases). Failover(handling leader failure) can be troublesome.
* multi-leader: more than one node accept writes. Hard to resolve write conflict, should be avoided.
* leaderless: no leader, read requests sent to multple nodes to resolve conflict values. No need for failover when quorum (number of read/write confirmation) properly configured.
## Replication Configuration
* Synchronous vs. asynchronous: full synchronous can block writes so is impractical; asynchronous is widely used but data among different nodes can be inconsistent.
## Consistency Guarantee
1. weakest guarantee: eventual consistency
2. stronger guarantees:
    * read-after-write consistency: a single user is guaranteed to see the change after writes from herself
    * monotonic reads: user will not see out-of-sequence data
    * consistent prefix reads: writes causally related to each other show in the order in which the writes occur
    
    Stronger guarantees can be made simpler (for the application layer) using **transactions** at the database layer
3. strongest: strong consistency (impractical)
## Replication Types
### Single-leader Replication
#### How is Data Replicated?
Write logs propagate from leader to followers. Logical (row-based) log (p.160) is the most common.
#### Node Outages
* followers: catch up changes using the log of the leader and itself
* leader: consuming from the new leader (aka. failover) with steps 1. figuring out leader has failed (with timeouts) 2. choosing a new leader (with most up-to-date data) 3. using the new leader
    * failover can go wrong in many ways: durability issue - new leader not caught up with old leader; inconsistency/reuse of primary keys; split brain(two leaders); timeout configuration. As a result, some teams still prefer performing failover manually.

### Multi-leader Replication
#### How?
a leader in each datacenter
#### Implementation 
CouchDB
#### Current Status
often implemented with external tools 
#### Pros & Cons
* Pros:
    * reduce latency (can write to more than one node)
    * better tolenrance of datacenter outages (no failover needed; other datacenters still work; can wait for the failed datacenter to come back)
    * better tolerance of network issues
* Cons:
    * write conflicts (autoincrementing keys, triggers, integrity consttraints)
#### Topologies
* circular
* star
* all-to-all (common, better fault-tolerance, one node failure does not impact other nodes likee circular and star topology do)
caveats: handling infinite loops

### Leaderless Replication
#### How it works?
Reads/Writes sent to multiple nodes; need w write confirmations and r read confirmations to gurantee read/write success otherwise returns an error. Replication happens asynchronously and without enforcing the order of writes.
#### Pro & Cons
* Pros:
    * tolerate conflicting oncurrent writes, network interruptions, and latency spikes
#### Implementation:
DynamoDB, Riak, Cassandra, Voldemort
#### Data Catch Up
* read repair - lazy repair on read
* anti-entropy process - background process
#### Read/write Quorums
As long as w(write confirmed nodes) + r(read queried nodes) > n(number of nodes), we expect to get an up-to-date value when reading (at least one of the n replicas you read stores the most recent successful write - "pigeon hole" principle) - in other words, r and w are the minimum number of votes required for the read or write to be valid
* Config:
    * w, r balancing: for write heavy - decrease the value of w and increase the value of r; for read heavy: vice versa
    * can pick w, r that w + r < n to achieve lower latency and higher availability (may see stale value)  
    * multi-datacenter: can configure n to be the number of nodes in all datacenters (Cassandra) or the number of local nodes (Riak)
* Limitations: can have edge cases where still see stale value 
* Sloppy Quorum: writes and reads require w and r successful responses by not necessarily from the designated n "home" nodes (eg. nodes in other datacenter; whether sloppy quorum is used can usually be configured in db imeplementations)
## Handling Write Conflicts
* What is concurrency: neither event A and B happens before each other or knows about each other. 
* How should concurrency be handled: when events are not concurrent, the later operation should oveerwrite the earlier, but if they are concurent, we need to resolve the conflict
* Asynchronous confict detection: to allow writes to multi nodes, asynchronous confict detection preferred over synchronous (otherwise better choose singe-leader replication)
* Approaches:
    * avoid conflict: (for multi-leader replication)all writes for a particular record go through the same leader (recommended)
    * converging to a consistent state:
        * LWW - last write wins (using timestamp etc.)
            * default implementation in many replicated databases
            * pros & cons:
                * (-) lost updates
        * replica with higher id wins
        * merge all values
        * record all values, use application code (or prompt user) to choose value (common)
            * steps:
                * Capturing conflicts (siblings) (p.187): using version number, merge when override, append when concurrent
                * Handling conflicts aka. merging siblings (concurrent values). Union, add deletion marker(tombstone) to removed item. Merging siblings in application code is error-prone, automatic conflict resolution is still under research
                * write conflicts with multi replica: version vectors - a version number per replica. Each replica increments its own version number, and also keeps track of the version numbers it has seen from each of the other replicas. This indicates which values to overwrite and which to keep as siblings.
            * types:
                * resolve conflict on write
                * resolve conflict on read
    * automatic conflict resolution (early stage, promising)






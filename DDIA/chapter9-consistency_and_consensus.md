# Chapter 9 - Consistency and Consensus
This chapter go through some examples of algorithms and protocols for building fault-tolerant distributed systems.
## How to build a fault-tolerant system?
Find some general-purpose abstractions with useful guarantees, implement them once, and then let applications rely on those guarantees (eg. using transaction implemented in db) so that they can ignore some of the problems with distributed systems
## What are the most important abstractions for distributed systems?
Concensus (getting all of the nodes to agree on something)
## Strong Consistency Guarantee - Linearizability
* linearizability is the strongest consistency guarantee: a linearizable register (object) behaves as if there's only a subgke ciot if tge data, and every operation appears to take effect atomically at one point in time.
* a useful guarantee, but few systems are actually linearizable due to its cost on availability and the fact that linearizability is slow. Weaker consistency guarantees (eg. ordering guarantees) is much faster.
* linearizable: all requests and responses can be arranged into a valid sequantial order (p.328)
* Problem: Eventual consistency is a very weak guarantee (doesn't guarantee when the replicas will converge). Stronger consistency models need to be explored. (pros: easier to use from applications perspective; cons: worse performance, less fault-tolerant)
* linearizability vs. serializability
    
    both mean something like "can be arranged in a sequential order
    * serializability:

        an isolation property of transactions: even through transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially, without any concurrency
    * linearizability:

        a regency guarantee on reads and writes of a **single object** (doesn't assume any transaction isolation)
    
    A database may provide both serializability and linearizability (aka. strict serializability). Of the three techiques of **serializable isolation**, actual serial execution and two-phase locking provide strict serializability, while serializable snapshot isolation does not.
* linearizability use cases
    * locking and leader election: single-leader replication system needs to ensure that there is indeed only one leader (no brain split); coordination services like Apache ZooKeeper are often used to implement distributed locks and leader election.
    * constraints and uniqueness guarantees: (eg. unique user id, bank account balance not negative)
    * cross-channel timing dependencies: communication went faster through one channel compared to the other, which is supposed to arrive first (eg. a webserver stores file and send message to resize the file through two different channels. However, the message queue received the resizing message first, when file hasn't been stored into storage yet)
* implementing linearizable systems
    * what replication methods can be made linearizable?
        1. single-leader replication: potentially linearizable if you make reads from the leader, or from synchronously updated followers; violates linearizability if a false leader (split brain) serves requests
        1. consensus algorithm: linearizable (has resemblance to single-leader replication with no split brain)
        1. multi-leader replication: not linearizable (can produce conflict writes)
        1. leaderless replication: probably not linearizable ("Last write wins based on time-of-day clocks with possible clock skews; sloppy quorums)
    * linearizability and quoraums

        It seems strict quorum reads and writes should be linearizable. However, when we have variable network delays (replication lag), it is possible to have race conditions. Therefore, it's safest to assume that a leaderless system with Dynamo-style replication does not provide linearizability.
* the cost of linearizability

    Applications that don't require linearizability can be more tolerant of network problems. (the CAP theorem)
    * the CAP theorem: consisteny(linearizability), availability, pick 1 out of 2 when partitioned
## A Weaker Consistency Guarantee - Ordering Guarantees
* only guarantees causality, ie. causally consistent; two events are ordered if they are causally reolated, but they are imcomparable if they are concurrent
* a weaker consistency than linearalizability: linearalizability guarantees total ordering (including events without causal relationships); in many cases, system that appear to require linearizability only really require causal consistency
    * advantage: the strongest possible consistency model that does not slow down due to network delays (like linearalizability does), and remains available in the face of network failures
* current status: promising direction, not yet made into production
* capturing causal dependencies:
    * how? - when a replica process an operation, it must ensure that all causally preceding operations have already been processed; if some preceding operation is missing, the later operation must wait until the preceding operation has been processed. It tracks causal dependencies across the entire database (not just for a single key) using version vectors. The version number from the prior operation is passed back to the database on a write in order to determine the causal ordering. (p.189)
    * methods:
        1. sequence number ordering: using sequence numbers or timestamps (something monotonically increasing) to order events that is consistent with causality. Lamport timestamps is usually used.
            * Lamport timestamps generate sequence numbers across nodes: each node has a unique id, and each node keeps a counter of the number of operations it has processed (counter, node ID). If you have two timestamps, the one with a greater counter value is the greater timestamp; if the counter values are the same, the one with the greater node ID is the greater timestamp. When a node sees a counter vlue greater than its own counter value (contained in a client request/response), it immediately increases its own counter to that maximum
            * disadvantage: not enough for implementing creating unique usernames (the total order of operations only emerges after the operation is completed; we need to know when that order is finalized - need total order broadcast)
        1. total order broadcast
        
## Distributed Transactions and Consensus
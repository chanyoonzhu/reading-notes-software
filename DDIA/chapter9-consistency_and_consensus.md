# Chapter 9 - Consistency and Consensus
This chapter go through some examples of algorithms and protocols for building fault-tolerant distributed systems.
## How to build a fault-tolerant system?
Find some general-purpose abstractions with useful guarantees, implement them once, and then let applications rely on those guarantees (eg. using transaction implemented in db) so that they can ignore some of the problems with distributed systems
## What are the most important abstractions for distributed systems?
Concensus (getting all of the nodes to agree on something)
## Strong Consistency Guarantee - Linearizability
* linearizability is the strongest consistency guarantee: a linearizable register (object) behaves as if there's only a single database, and every operation appears to take effect atomically at one point in time.
* problem it solves: Eventual consistency is a very weak guarantee (doesn't guarantee when the replicas will converge). Stronger consistency models need to be explored. (pros: easier to use from applications perspective; cons: worse performance, less fault-tolerant)
* a useful guarantee, but few systems are actually linearizable due to its cost on availability and the fact that linearizability is slow (especially with large network delays). Weaker consistency guarantees (eg. ordering guarantees) is much faster.
* linearizable: all requests and responses can be arranged into a valid sequantial (total) order (p.328 figure 9-4)
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
* sequential consistency (timeline consistency): slightly weaker than linearizability. It only guarantees linearizable writes, not linearizable reads (good enough for assigning unique username)
## A Weaker Consistency Guarantee - Ordering Guarantees
* only guarantees causality, ie. causally consistent; two events are ordered if they are causally reolated, but they are imcomparable if they are concurrent
* a weaker consistency than linearalizability: linearalizability guarantees total ordering (including events without causal relationships); in many cases, system that appear to require linearizability only really require causal consistency
    * advantage: the strongest possible consistency model that does not slow down due to network delays (like linearalizability does), and remains available in the face of network failures
* current status: promising direction, not yet made into production
* pros & cons
    (+) does not have the coordination overhead of linearizability
    (+) much less sensitive to network problems
    (-) some things like unique username assignment cannot be implemented (does not know whether another node is concurrently in the process of registering the same name). (need to achieve **consensus**)
* capturing causal dependencies:
    * how? - when a replica process an operation, it must ensure that all causally preceding operations have already been processed; if some preceding operation is missing, the later operation must wait until the preceding operation has been processed. It tracks causal dependencies across the entire database (not just for a single key) using version vectors. The version number from the prior operation is passed back to the database on a write in order to determine the causal ordering. (p.189)
    * methods:
        1. sequence number ordering: using sequence numbers or timestamps (something monotonically increasing) to order events that is consistent with causality. Lamport timestamps is usually used.
            * Lamport timestamps generate sequence numbers across nodes: each node has a unique id, and each node keeps a counter of the number of operations it has processed (counter, node ID). If you have two timestamps, the one with a greater counter value is the greater timestamp; if the counter values are the same, the one with the greater node ID is the greater timestamp. When a node sees a counter vlue greater than its own counter value (contained in a client request/response), it immediately increases its own counter to that maximum
            * disadvantage: not enough for implementing creating unique usernames (the total order of operations only emerges after the operation is completed; we need to know when that order is finalized - need total order broadcast)
        1. total order broadcast
            * definition: (delivered exactly once, in the same order)
                * reliable delivery: no messages are lost; if a message is deliverd to one node, it is delivered to all nodes
                * totally ordered delivery: messages are delivered to every node in the same order
            * implementation: ZooKeeper
            * used for: 
                * database replication: if every message represents a write to the database, and every replica processes the same writes in the same order, then the replicas will remain consistent with each otther (aka. state machiine replication)
                * implementing serializable transactions
                * implementing lock service that provides fencing tokens
            * implementing linearizable storage using total order broadcast

                If you have total order broadcast, you can build linearizable storage on top of it (eg. unique usernames)
                * how? 
                    1. append a message to the log indicating the username you're claining
                    1. read the log, and wait for the message you appended to be delivered back to you
                    1. check for any messages claiming the username that you want. If the first message is your own, then the claiming is successful, otherwise abort. (this is based on the total ordering of messages)
            * implementing total order broadcast using linearizable storage
                * how? 
                    1. for a message to be sent through total order broadcast, you increment-and-get the linearizable integer, and then attach the value as a sequence number to the message.
                    1. send the message to all nodes (resending lost messages), and the recipients will deliver the messages consecutively by sequence number (if a number in the middle is missing, the node will wait)
                * hard to have a linearizable storage when: network connections to the node keeping the linearizable integer are interrupted; restoring the value when that node fails
## Distributed Transactions and Consensus
* The most important problem in distributed system: get several nodes to agree on something irevocably.
* definition of concensus problem: one or more nodes may propose values, and the consesus algorithm decides on one of those values. 
* equivalent problems that can be reduced to concensus problems:
    1. linearizable comprare-and-set registers: the register(object/variable) needs to atomically decide whether to set its value, based on whether its current value equals the parameter given in the operation
    1. atomic transaction commit: a database must decide whether to commit or abourt a distributed transaction
    1. total order broadcast: the messaging system must decide on the order in which to deliver messages
    1. locks and leases: when several clients are racing to grab a lock or lease, the lock decides which on successfully acquire it
    1. membership/coordination service: given a failuree detector (eg., timeouts), the system must decidd which nodes are alive, and which should be considered dead because their sessions timed out
    1. uniqueness constraint: when several transactions concurrently try to create conflicing records with the same key, the constraint must decide which one to allow and which should fail with a constraint violation
* a consensus algorithm must satisfy the following properties:
    1. uniform agreement: no two nodes dicide differently
    1. integrity: no node decides twice
    1. validity: if a node decides value v, then v was proposed by some node
    1. termination: every node that does not creash eventually decides some value
    * pros & cons:
        (+)a concensus algorithms are a huge break through for distributed systems. They can implement linearizable atomic operations in a fault-tolerant way.
        (-) most consensus algorithm assume a fixed set of nodes participating in voting; blocked on synchronous replication when voting on proposals; sensitive to network problem (uses timeouts to detect node failure)
* Use cases:
    * leader election: no split brain
    * atomic commit: a transaction either commit or rolled back on all nodes
* Algorithms:
    1. two-phase commit (2PC)
        * most common way of solving atomic commit in a distributed system; implemented in various databases, messaging systems, and application servers. Not a very good concensus algorithm - weak fault-tolerance when coordinator fails.
        * when should a node in a distributed system commit? Only when it's certain that tall other nodes in the transaction are also going to commit
        * implementation: most "NoSQL distributed datastores do not support distributed atomic transactions, but various clustered relational systems do.
        * steps:
            1. reading and writing data on multiple database nodes as normal
            1. the coordinator (in the application or a dedicated service) sends a prepare request to each of the participating nodes, asking them whether they're able to commit (phase 1)
            1. if all participants answer "yes", the coordinator sends out a committ request to nodes to commit, otherwise sends an abort request. (phase 2)
        * details: each read/write sent to nodes are tracked with a globally unique transaction id (managed by coordinator) if abort is needed. Once a node answer "yes", it must commit the transaction in phase 2 if the coordinator chooses to do so (even if node crashes, it will commit after it comes back). Once a coordinator decides all nodes should commit, all nodes must commit in spite of request fail or timeout (the coordinator keeps retrying until successful). If the coordinator crashes, the participants can do nothing but wait. The coordinator must write its commit or abort decision to a transaction log on disk before sending out final requests to helper recover from crash.
        * pros & cons:
            (+) result is similar as a regular single-node atomic commit
            (-) does not satisfy the "Termination" properties of a consensus algorhm: a blocking commit when the coordinator crashes and all nodes waiting for it to recover at phase 2. (performance penalty - database cannot release those locks until the transaction commits or aborts and no other transaction can modify those rows until then; many cloud services choose not to implement distributed transactions due to this reason)
            (-) single point of failure if the coordinator runs only on a single node
            (-) coordinator log makes the application not stateless (need to store the transaction log in disk, like a database)
        * a standard for implementation across heterogeneous technologies (participants of 2PC use two or more different technologies) - XA transactions
            * a C API for interfacing with a transaction coordinator.
            * the coordinator in application calls the WA API to find out whether an operation should be part of a distributed transaction, and ask the participants to prepare, commit, or abort
    * fault-tolerant consensus algorithms
        * the best known fault-tolerant consensus algorithm are: Viewstamped Replication, Paxos, Raft, and Zab
        * are in fact total ordering broadcast (which is equivalent to repeated rounds of consensus)
        * elects a "leader" in each epoch using an incremented epoch number to track votes. The leader if voted from a quorum of nodes
        * biggest difference from 2PC: cencus is arrived from a majority of nodes (quorum) where as in 2PC, concensus need to be obtained from all nodes
* ZooKeeper - a service implementing concensus algorithm
    * what is ZooKeeper used for: users "outsource" some of the work of coordinating nodes to SooKeeper
    * use cases:
        1. choose a leader (for databases, job schedulers, etc.)
        1. deciding which partition to assign to which node
        1. service discovery (actually not requuires consensus that much, DNS can do the job)
        1. membership services: determines which nodes are currently active and live members of a cluster

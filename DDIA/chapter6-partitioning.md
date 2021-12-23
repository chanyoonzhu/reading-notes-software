# Chapter 6 - Partitioning
## Why partitioning:
Having multiple copies of the same data on different loads is not sufficient for large dataset or high query **throughput**. Data needs to be broken into partitions(shards).
## Intro
* each data belongs to exactly one partition
* used alongside replication (chapter 5)
## Goal
spread data and the query **evenly** across nodes, avoid hot spot
## Strategies
* by key range
    * pro: range scan is easy
    * con: hotspots (eg. timestamp all go to one partition on a particular day)
* by hash of key
    * pro: distribute data evenly
        * skewed data can still happen (eg. celebrity record), can split contents by storing in multiple keys appended with a random number (but increase application logic on read)
    * con: cannot do range scan
        * can use combined key (hashed key + range key): Cassandra & DynamoDB
## Secondary Indexes
* by document (local secondary index)
    * pro: fast write
    * con: slow read - needs to combine results from all partitions if querying a secondary key
* by term (global secondary index)
    * pro: fast read
    * con: slow write
## Rebalancing Partitions
### When?
when changes call for data and requests to be moved from one node to another.
### Goal:
* load is shared fairly
* service continues while rebalancing
* only move data that needs to be moved
### Strategies
* fixed number of partitions
    * implementation: Elasticsearch, Couchbase, Riak, Voldemort
    * pros & cons:
        * (+) simple logic
        * (-) no partition splitting
        * (-) number of partitions can't change - hard to choose the number of partitions that allows for future growth (rebalancing and recovery becomes expensive if the number of partitions is small/ each partition is too large, too much overhead if each the number of partitions is too large)
* dynamic partitioning
    * implementation: MongoDB, HBase, RethinkDB
    * how: a partition is split into two when it exceed a configured size, one half is transferred to another node to balance the load. (The number of partitions is proportional to the size of the dataset)
    * pros & cons
        * (+) the number of partitions adapts to the tottal data volume
        * (-) only one node is active when data size is small (can be avoided by pre-splitting)
* Partitioning proportionally to nodes
    * implementation: Cassandra
    * pros & cons
        * (+) the size of partitions is stable
        * (-) only one node is active when data size is small (can be avoided by pre-splitting)

Manual intervention: fully automated rebalancing can be unpredictable, the process can overload the network or the nodes and harm the performance of other requests while rebalancing is in progress; better to have a human in the loop for rebalancing

## Request Routing
### Strategies
* a dedicated routing tier (utilizing a separate coordination service such as Zoo-Keeper) that determines which node the request should be forwarded to (cluster management)
    * implementations: HBase, Kafka
* routing info kept on all nodes, the node accepting the request tells where the request should be routed to.
    * implementations: Cassandra, Riak
    * pros & cons:
        * (+) single-point-of-failure on the coordination service
        * (-) more complexity in the nodes





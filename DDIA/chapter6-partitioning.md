# Chapter 6 - Partitioning

---

## Why Partitioning?

Having multiple copies of the same data on different nodes is not sufficient for large datasets or high query **throughput**. Data needs to be broken into **partitions (shards)**.

---

## Introduction

- Each data belongs to exactly one partition
- Used alongside replication (Chapter 5)

---

## Goal

Spread data and the query **evenly** across nodes, avoid hot spots.

---

## Strategies

### By Key Range

**Pros:**
- ✅ Range scan is easy

**Cons:**
- ❌ Hotspots (e.g., timestamp all go to one partition on a particular day)

### By Hash of Key

**Pros:**
- ✅ Distribute data evenly
  - Skewed data can still happen (e.g., celebrity record), can split contents by storing in multiple keys appended with a random number (but increase application logic on read)

**Cons:**
- ❌ Cannot do range scan
  - Can use **combined key** (hashed key + range key): **Cassandra** & **DynamoDB**

---

## Secondary Indexes

### By Document (Local Secondary Index)

**Pros:**
- ✅ Fast write

**Cons:**
- ❌ Slow read - needs to combine results from all partitions if querying a secondary key

### By Term (Global Secondary Index)

**Pros:**
- ✅ Fast read

**Cons:**
- ❌ Slow write

---

## Rebalancing Partitions

### When?

When changes call for data and requests to be moved from one node to another.

### Goal

- Load is shared fairly
- Service continues while rebalancing
- Only move data that needs to be moved

### Strategies

#### Fixed Number of Partitions

**Implementation**: **Elasticsearch**, **Couchbase**, **Riak**, **Voldemort**

**Pros & Cons:**
- ✅ Simple logic
- ❌ No partition splitting
- ❌ Number of partitions can't change - hard to choose the number of partitions that allows for future growth
  - Rebalancing and recovery becomes expensive if the number of partitions is small/each partition is too large
  - Too much overhead if the number of partitions is too large

#### Dynamic Partitioning

**Implementation**: **MongoDB**, **HBase**, **RethinkDB**

**How**: A partition is split into two when it exceeds a configured size, one half is transferred to another node to balance the load. (The number of partitions is proportional to the size of the dataset)

**Pros & Cons:**
- ✅ The number of partitions adapts to the total data volume
- ❌ Only one node is active when data size is small (can be avoided by pre-splitting)

#### Partitioning Proportionally to Nodes

**Implementation**: **Cassandra**

**Pros & Cons:**
- ✅ The size of partitions is stable
- ❌ Only one node is active when data size is small (can be avoided by pre-splitting)

> **Note**: Manual intervention - fully automated rebalancing can be unpredictable, the process can overload the network or the nodes and harm the performance of other requests while rebalancing is in progress; better to have a human in the loop for rebalancing.

---

## Request Routing

### Strategies

#### 1. Dedicated Routing Tier

Utilizing a separate coordination service such as **ZooKeeper** that determines which node the request should be forwarded to (cluster management).

**Implementations**: **HBase**, **Kafka**

#### 2. Routing Info on All Nodes

The node accepting the request tells where the request should be routed to.

**Implementations**: **Cassandra**, **Riak**

**Pros & Cons:**
- ✅ No single-point-of-failure on the coordination service
- ❌ More complexity in the nodes

---

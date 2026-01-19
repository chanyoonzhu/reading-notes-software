# Chapter 10 - Batch Processing

---

## Different Types of Systems

1. **Services (online systems)**: Request/response; response time and availability is important
2. **Batch processing systems (offline systems)**: Scheduled to run periodically to process a large amount of input data; throughput is important
3. **Stream processing systems (near-real-time systems)**: Operates on events shortly after they happen; have lower latency than batch systems

---

## Batch Processing with Unix Tools

### Importance

The ideas and lessons from Unix are carried over to large-scale, heterogeneous distributed data systems (e.g., MapReduce).

### Important Characteristics of Unix Tools

1. **Immutable inputs**
2. **Chaining of commands**
3. **Sort** (vs. in-memory aggregation): Handles larger-than-memory datasets by spilling to disk, and automatically parallelizes sorting across multiple CPU cores
4. **A unique interface**: A file (descriptor)
5. **Separation of (input/output) logic and wiring**: The program doesn't have to know where the input is coming from and where the output is going to - all it uses is `stdin` and `stdout`
6. **Transparency and experimentation**: Immutable input; debugging tool (`less`); can write the output to file

### Example

```bash
cat /var/log/nginx/access.log  |
awk '{print $7}'  |
sort  |
uniq -c  |
sort -r -n  |
head -n 5
```

### Hadoop and Unix

Hadoop is somewhat like a distributed version of Unix, where **HDFS** is the filesystem and **MapReduce** is a quirky implementation of a Unix process (which happens to always run the sort utility between the map phase and the reduce phase).

---

## Map-Reduce

### Overview

A programming framework with which you can write code to process large datasets in a distributed filesystem (it is a bit like Unix tools, but distributed across potentially thousands of machines).

### Input/Output

Files on a distributed share-nothing filesystem (e.g., **HDFS**, or Hadoop Distributed File System).

### Implementation

Various open source data systems: **Hadoop**, **CouchDB**, and **MongoDB**.

### Components

- **Name node**: Keeps track of which file blocks are stored on which machine
- **Machines**: Store file blocks (with redundancy)

### Job Execution

**Stages:**

1. **Mapper**: Called once for every input record to extract the key and (transformed) value from each individual input record, and output key and value pair in sorted order (sorted by key)
2. **Reducer**: Collects all the values belonging to the same key and produce output by iterating over the collections of values

> **Note**: Why does the output of mapper have to be sorted? Because all the same keys could become adjacent to each other in the reducer input.

### Pros & Cons

**Pros:**
- ✅ Separates the physical network communication aspects of the computation (getting the data to the right machine) from the application logic
- ✅ Shields the application code from having to worry about partial failures (e.g., the crash of another node); it transparently retries failed tasks without affecting the application logic
- ✅ Easy to debug: inputs are immutable and outputs from failed tasks are discarded

**Cons:**
- ❌ Can only start when all tasks in the preceding jobs have completed (job skew when one node is flooded with data)
- ❌ Mappers are often redundant if the result of a reducer can be immediately chained to the next reducer
- ❌ Intermediate state files (materialized view) are stored and replicated across several nodes, which is often overkill for temporary data

### Chained Workflows

The range of problems one can solve with a single MapReduce job is limited. Thus, it's common for MapReduce jobs to be chained together into workflows (the output of one job becomes the input to the next job). 

**Workflow schedulers** (e.g., **Oozie**, **Azkaban**, etc.) are developed for Hadoop to handle dependencies between job executions, and higher-level tools for Hadoop (e.g., **Pig**, **Hive**, etc.) also set up workflows of multiple MapReduce stages that are automatically wired together appropriately.

### Bringing Related Data to the Same Place with Join Algorithms

The key, value pair of the same key is always routed to the same reducer.

#### 1. Reduce-Side Joins

Since the mapper output is sorted by key, the reducer can merge together the sorted lists of records easily (keeping all values of a key in memory at any one time without having to make any requests over the network).

#### 2. Reduce-Side Grouping

By setting up the mappers so that the key-value pairs they produce use the desired grouping key.

#### 3. Map-Side Joins

Can be used when you have certain assumptions about your data.

- **Broadcast hash joins**: Each mapper for a partition reads the entirety of the small input and do the join
- **Partitioned hash joins**: When inputs (that need a join) are partitioned in the same way, then the hash join approach can be applied to each partition independently
- **Map-side merge join**: When inputs are partitioned in the same way and also sorted based on the same key, then the mapper can read from both inputs incrementally and match records with the same key

#### Reducer-Side or Map-Side Join?

Determined by what output is needed: the output of a reduce-side join is partitioned and sorted by the join key, whereas the output of a map-side join is partitioned and sorted in the same way as the large input.

#### Handling Skew

**Why**: Hotkeys all go to the same reducer, since the subsequent jobs must wait for the slowest reducer to complete before they can start, the reducer with the hotkey becomes a bottleneck.

**How**: **Sharded join** - spreads the work of handling the hot keys over several reducers.

### Batch Workflows Use Cases

1. Building search indexes (over a fixed set of documents)
2. Building machine learning systems such as classifiers, outputting results to a queryable database (some key-value stores support building database files in MapReduce jobs)

### Comparing Hadoop to Distributed Databases (MPP)

The biggest difference is that distributed databases focus on parallel execution of analytic SQL queries on a cluster of machines, while the combination of MapReduce and a distributed filesystem provides something much more like a general-purpose operating system that can run arbitrary programs.

#### Diversity of Storage

| | **Hadoop** | **Distributed Databases** |
|-----------| ----------- | ----------- |
| **Storage** | Data of any format | Data conform to a schema |
| **Processing models** | A general model | SQL query |
| **Handling of faults** | Retry the individual task (which failed) only | Abort the entire query, retry as a whole |
| **Memory/disk utilization** | Prefer to write to disk for fault tolerance and on the assumption that the data is large | Prefer to keep as much data as possible in memory |

**Advantages of Hadoop:**

1. Raw data is better: allows for several types of different transformations
2. Not all kinds of processing can be sensibly expressed as SQL queries
3. More fault-tolerant especially when batch tasks can be preempted by other much more urgent tasks running on the same machine

---

## Beyond MapReduce

Besides MapReduce, other tools may be more appropriate for expressing a computation and are sometimes orders of magnitude faster for some kinds of processing. MapReduce treats each task in a workflow as a single job with time and storage overhead. Instead, these new systems handle an entire workflow as one job (just like Unix pipelines, processes connected by a Unix pipe are started at the same time, with output being consumed as soon as it is produced).

### 1. Dataflow Engines

**Implementations**: **Spark**, **Tez**, and **Flink**.

**Components**: **Operator** (a generalization of mapper and reducer) that can:

1. Repartition and sort records by key
2. Repartition key without sorting
3. Broadcast hash joins

Operators are arranged in a job as a **directed acyclic graph (DAG)**.

**Advantages:**

- ✅ Expensive work such as sorting only need to be performed in places where it is actually required, rather than always happening by default between every map and reduce stage
- ✅ No unnecessary map tasks if mapper not needed for a subtask
- ✅ The scheduler has an overview of what data is required where (since all joins and data dependencies in a workflow are explicitly declared), so it can make locality optimizations
- ✅ Intermediate state are kept in memory or written to local disk, which requires less I/O than writing it to HDFS (which involves replication)
- ✅ Operators can start executing as soon as their input is ready, no need to wait for the entire preceding stage to complete
- ✅ Existing JVM processes can be reused to run new operators, reducing startup overheads compared to MapReduce that launches a new JVM for each task

**Fault-Tolerance Strategy** (when there's no materialized views as in MapReduce):

- Recomputed from other data that is still available
- In the case of nondeterministic, operators kills the downstream operators as well, and run them again on the new data

### 2. Graphs and Iterative Processing

**Batch processing an entire graph**

**Difference**: The processing needs to be repeated until certain condition is made (e.g., no un-traversed edges), this "repeating until done" idea cannot be expressed in plain MapReduce.

**Solution**: **Pregel model** - a vertex remembers its state in memory from one iteration to the next, so the function only needs to process new incoming messages. The job completes if no messages are being sent in some part of the graph.

**Fault-Tolerance**: Achieved by periodically checkpointing the state of all vertices at the end of an iteration.

**Disadvantage**: The graph is often simply partitioned by an arbitrarily assigned vertex ID without any attempt to group related vertices together (which results in a lot of cross-machine communication overhead). Efficiently parallelizing graph algorithms is an area of ongoing research.

---

## High-Level APIs and Languages

Higher-level languages and APIs such as **Hive**, **Pig**, **Cascading**, and **Crunch** became popular because programming MapReduce jobs by hand is quite laborious. In addition, they have the benefit of being able to move from MapReduce to the new dataflow execution engine without having to rewrite job code.

### The Move Toward Declarative Query Languages

**Why**: The choice of join algorithm (map-side/reduce-side join) can make a big difference to the performance of a batch job, and it is nice not to have to understand all the various join algorithms if joins can be specified in a declarative way and let the query optimizer decide how they can be best executed.

**Result**: By incorporating declarative aspects in their high-level APIs, and having query optimizers that can take advantage of them during execution, batch processing framework begin to look more like distributed databases (MPP).

---

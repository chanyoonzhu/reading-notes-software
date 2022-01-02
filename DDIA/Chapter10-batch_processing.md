# Chapter 10 - Batch Processing
## Different Types of Systems
    1. Services (online systems): request/response; response time and availability is important
    1. batch processing systems (offline systems): scheduled to run periodically to process a large amount of input data; throughput is important
    1. stream processing systems (near-real-time systems): operates on events shortly after they happen; have lower latency than batch systems
## Batch Processing with Unix Tools
* importance: the ideas and lessons from Unix are carried over to large-scale, heterogeneous distributed data systems (eg. MapReduce)
* important characteristics of Unix tools:
    1. immutable inputs
    1. chaining of commands
    1. sort (vs. in-memory aggregation): hanels larger-than-memory datasets by spilling to disk, and automatically parallelizes soorting across multiple CPU cores.
    1. a unique interface: a file (descriptor)
    1. separation of (input/output) logic and wiring: the program doesn't have to know where the input is coming from and where the output is going to - all it uses is stdin and stdout
    1. transparency and experimentation: immutable input; debugging tool (less); can write the output to file

    example:
    ```
    cat /var/log/nginx/access.log  |
    awk '{print $7}'  |
    sort  |
    uniq -c  |
    sort -r -n  |
    head -n 5  |
    ```
* Hadoop and Unix: Hadoop is somewhat like a distributed version of Unix, where HDFS is the filesystem and MapReduce is a quirky implementation of a Unix process (which happens to always run the sort utility between the map phase and the reduce phase)
## Map-Reduce
* a programming framework with which you can write code to process large datasets in a distributed filesystem (it is a bit like Unix tools, but distributed across potentially thousands of machines).
* input/output: files on a distributed share-nothing filesystem (eg. HDFS, or Hadoop Distributed File System)
* implementation: various open source data systems: Hadoop, CouchDB, and MongoDB
* components:
    * name node: keeps track of which file blocks are stored on which machine
    * machines: stores file blocks (with redundency)
* job execution:
    * stages:
        1. mapper: called once for every input record to extract the key and (transformed) value from each individual input record, and output key and value pair in sorted order (sorted by key)
        1. reducer: collects all the values belonging to the same key and produce output by iterating over the collections of values

        note: why the output of mapper has to be sorted? Because all the same keys could become adjacent to each other in the reducer input
* pros & cons:
    * (+) separets the physical network comminication aspects of the computation (getting the data to the right machine) from the application logic
    * (+) shields the application code from having to worry about partial failures (eg. the crash of another node); it transparently retries failed tasks without affecting the application logic
    * (+) easy to debug: inputs are immutable and outputs from failed tasks are discarded
    * (-) can only start when all tasks in the preceding jobs have completed (job skew when one node is flooded with data)
    * (-) mappers are often redundant if the result of a reducer and be immediately chained to the next reducer
    * (-) intermediate state files (materialized view) are stored and replicated across several nodes, which is often overkill for temporary data
* chained workflows

    The range of problems one can solve with a single MapReduce job is limited. Thus, it's common for MapReduce jobs to be chained together into workflows (the output of once job becomes the idnput to the next job). Workflow schedulers (eg. Oozie, Azkaban, etc.) are developed for Hadoop to handle dependencies between job executions, and higher-level tools for Hadoop (eg. Pig, Hive, etc.) also set up workflows of multiple MapReduce stages that are automatically wired together appropriately.
* bringing related data to the same place with join algorithms
    
    the key, value pair of the same key is always routed to the same reducer
    1. reduce-side joins
        * since the mapper output is sorted by key, the reducer can merge together the sorted lists of records easily (keeping all values of a key in memory at any one time without having to make any requests over the network)
    1. reduce-side grouping
        * by setting up the mappers so that the key-value pairs they produce use the desired grouping key
    1. map-side joins
        * can be used when you have certain assumptions about your data.
            1. broadcast hash joins: each mapper for a partition reads the entirety of the small input and do the join
            1. partitioned hash joins: when inputs (that need a join) are partitioned in the same way, then the hash join approach can be applied to each partition independently.
            1. map-side merge join: when inputs are partitioned in the same way and also sorted based on the same key, then the mapper can read from both inputs incrementally and match records with the same key
    * reducer-side or map-side join?

        determined by what output is needed: the output of a reduce-side join is partitioned and sorted by the join key, whereas the output of a map-side join is partitioned and sorted in the same way as the large input.

    * handling skew:
        * why: hotkeys all go to the same reducer, since the subsequent jobs must wait for the slowest reducer to complete before they can start, the reducer with the hotkey becomes a bottleneck.
        * how: sharded join - spreads the work of handling the hot keys over several reducers
* batch workflows use cases:
    1. building search indexes (over a fixed set of documents)
    1. building machine learning systems such as classifiers, outputing results to a querable database (some key-value stores support building database files in MapReduce jobs)
* comparing Hadoop to distributed databases (MPP)
    
    The biggest difference is that distributed databases focus on parallel execution of analytic SQL queries on a cluster of machines, while the combination of MapReduce and a distributed filesystem provides something much more like a general-purpose operating system that can run arbitrary programs.
    * diversity of storage: 

        || Hadoop | distributed databases |
        |-----------| ----------- | ----------- |
        |storage| data of any format | data conform to a schema |
        |processing models| a general model | SQL query |
        |handling of faults|retry the individual task (which failed) only|abort the entire query, retry as a whole|
        |memory/disk utilization|prefer to write to disk for fault tolerance and on the assumption that the data is large|prefer to keep as much data as possible in memory|

        * advantages of Hadoop:
            1. row data is better: allows for several types of different transformations
            1. not all kinds of processing can be sensibly expressed as SQL queries
            1. more fault-tolerent especially when batch tasks can be preempted by other much more urgent tasks running on the same machine

## Beyond MapReduce
* Besides MapReduce, other tools may be more appropriate for expressing a computation and are sometimes orders of magnitude faster for some kinds of processing. MapReduce treats each task in a workflow as a single job with time and storage overhead. Instead, these new systems handle an entire workflow as one job (just like Unix pipelines, processes connected by a Unix pipe are started at the same time, with output being consumed as soon as it is produced)
1. dataflow engines
    * implementations: Spark, Tex, and Flink
    * components: operator (a generalization of mapper and reducer) that can -
        1. repartition and sort records by key
        1. repartition key without sorting
        1. broadcast hash joins

        operators are arranged in a job as a directed acyclic graph(DAG)
    * advantages:
        * expensive work such as sorting only need to be performed in places where it is acttually required, rather tthan always happening by default between every map and reduce stage
        * no unnecessary map tasks if mapper not needed for a subtask
        * the scheduler has an overview of what data is required where (since all joins and data dependencies in a workflow are explicitly declared), so it can make locality optimizations
        * intermediate state are kept in memory or written to local disk, which requires less I/O than writing it to HDFS (which involves replication)
        * operators can start executing as soon as their input is ready, no need to wait for the entire preceding stage to complete
        * existing JVM processes can be reused to run new operators, reducing startup overheads compared to MapReduce that luanches a new JVM for each task
    * fault-tolerance strategy (when there's no materialized views as in MapReducer)

        * recomputed from other data that is still available
        * in the case of nondeterministic, operators kills the downstream operators as well, and run them again on the new data
1. graphs and iterative processing
    * batch processing an entire graph
    * difference: the processing needs to be repeated until certain condition is made (eg. no un-travered edges), thiis "repeating until done" idea cannot be expressed in plain MapReduce
    * solution: Pregal model - a vertex remembers its state in memory from one iteration to the next, so thee function only needs to process new incoming messages. The job completes if no messages are being sent in some part of the graph
    * fault-tolerance: achieved by periodically checkpointing the state of all vertices at the end of an iteration
    * disadvantage: the graph is often simply partitioned by an arbitrarily assigned vertex ID without any attempt to group related vertices together (which results in a lot of cross-machine communicattion overhead). Efficiently parallelizing graph algorithms is an area of ongoing research
## high-level APIs and languages
* higher-level languages and APIs such as Hive, Pig, Cascading, and Crunch became popular because programming MapReduce jobs by hand is quite laborious. In addition, they have the benefitt of being able to move from MapReduce to the new dattaflow executtion engine without having to rewrite job code.
* the move toward declarative query languages:
    * why: the choice of join algorithm (map-side/reduce-side join) can make a big difference to the performance of a batch job, and it is nice not to have to understand all the various join algorithms if joins can be specified in a declarative way and let the query optimizer to decide how they can be best executted
    * result: by incorporating declarative aspects in their high-level APIs, and having query optimizers that can take advantage of them during execution, batch processing framework begin to look more like distributed databases (MPP)

    

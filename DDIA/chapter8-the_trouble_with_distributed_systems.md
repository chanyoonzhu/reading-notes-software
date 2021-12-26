# Chapter 8 - The Trouble with Distributed Systems
## Anything that can go wrong will go wrong
Working with a distributed system is fundamentally diffeerent from writing software on a single computer. The task as engineers is to build systems that meet the guarantees that users are expecting, in spite of everything going wrong.
## Faults and Partial Failurees
An individual computer with good software is usually either fully functional or entirely broken (due to hardware faults), but not something in between. However, in distributed systems, nondeterminism and the possibility of partial failures makes it hard to work with. We need to build a reliable system from unreliable components.
## Problems with Distributed Systems
Detecting faults is hard (usually uses timeouts by can't distinguish between network and node failures). Tolerate a fault is not easy either (the system shares nothing and relies on messages).
* problems: 
    1. Unreliable Networks
        * issue: the internet and most internal networks in datacenters (often Ethernet) are asynchronous packet networks with unbounded delays (the network gives no guarantees as to when the packet will arrive, or whether it will arrive at all)
            * request is lost 
            * request is queued and will be delivered later
            * the remote node fails
            * the remote node temporarily stops responding
            * the remote node's response has been lost
            * the remote node's response has been delayed
        * solution: timeout (still don't know whether the request is processed or not)
        * network faults in practice: all things can go wrong, your software needs to be able to recover from them (eg. cluster could become deadlocked and permanently unable to serve requests, even when the network recovers, or it could delete all of your data)
        * detecting faults:
        the uncertainty about the network makes it difficult to tell whether a node is working or not. If you want to be sure that a request was successful, you need a positive response from the application itself. In general, you have to assume that you will get no response at all. You can retry a few times (at the application level), wait for a timeout to elapse, and eventually declare the node dead if it times out.
        * timeouts and unbounded delays:
        How long the timeout should be is not a simple question. The problems with low timeouts are: retry may end up processing the same action twice; additional load on the nodes and the network when transferring work to other nodes.
            * network congestion and queueing scenarios: 
                * packet is dropped when the network switch queue fills up
                * a request is queued by the operating system until the CPU core is ready
                * in virtualized environments, the request is queued by the virtual machine monitor and paused for a short time while another virtual machine uses a CPU core
                * TCP has flow control of sending requests
                * TCP implements timeouts and automatic retry
            * network congestion is the worst when system is highly utilized, by yourself or your "noisy neighbor" who shares the resources with you.
            * how to choose timeouts:
                * by experiments: measure round-trip times over an extended period over many machines
                * automatic timeouts adjustment (eg. Phi Accrual failure detector; TCP retransmission timouts)
        * TCP vs. UDP
        UDP is a good choice in situations where delayed data is worthless (eg. videoconferencing) because it does not perform flow control and packet retransmission like TCP does.
        * synchronous vs. asynchronous networks (latency guarantee vs. resource utilization)
            1. synchronous - circuit network: a fixed amount of reserved bandwidth which nobody else can use;
                pros: bounded delay (more predictable)
                cons: not optimized for bursty traffic (reduced utilization)
            1. async - TCP: the packets opportunistically use whatever network bandwidth is available
                pros: optimized for bursty traffic
                cons: unbounded delay
    1. Unreliable Clocks
        * clocks are used for:
            1. measureing durations (eg.timeouts, response time, throughput, how long did the user spend on the site)
            1. measuring points in time (eg. when is the article published, what time to send the notification, when does the cache expire, what is the timestamp of an error message in the log file)
        * problem: 
            1. in a distributed system, it's difficult to determine the order in which things happened since message need to travel across the network from one machine to another
            1. clocks on each machine are not synchronized
        * monotonic vs. time-of-day clocks
            1. time-of-day clocks: returns the current date and time according to some calendar; synchronized with the NTP server;
            1. monotonic clocks: measuring a duration, such as a timeout or a service's response time; guaranteed to always move forward;

            In a distributed system, using a monotonic clock for measuring elapsed time is usually better since it doesn't assume any synchronization between different nodes' clocks.
        * clock synchronization and accuracy (for time-of-day clocks): Unfortunately, synchronization methods aren't that reliable and accurate:
            * clock drifts due to temperature change of the machine
            * clock refuses to synchronize when differs too much from an NTP server
            * clock seeing time going back when NTP resets itt
            * a node is accidentally firewalled off from NTP server
            * NTP synchronization can only be as good as the network delay
            * NTP goes wrong or misconfigured.
            * leap seconds issue
            * in virtual machines, the hardware clock is virtualized which is unreliable
            * users deliberately set their hardware clock to incorrect date and time to circumvent timing limitations
        * relying on synchronized clocks:
            If you use software that reuires synchronized clocks, it is essential that you also carefully monitor the clock offsets between all the machines. Here are scenarios where clocks can be misused:
            * timestamps for ordering events: when last-write-wins strategy is used to resolve conflicts, a write(B) that is causally later than another write(A) may ended up having an earlier timestamp due to clock synchronization issues. Better using **logical clocks** that tracks the relative ordering of events
            * clock readings have a confidence interval: it's better to think of a clock reading as a range of times, within a confidence interval (eg. a system is 95% confident that the time now is between 10.2 and 10.5 seconds past the minute); most systems don't expose this uncertainty except for Google's TrueTime API in Spanner
            * synchronized clocks for global snapshots: using confidence level reading to determine the order of transaction ids among multiple nodes for snapshot isolation implementation (where an monotonically increasing transaction ID is used)
    1. process pauses：
        * A node in a distributed system must assume that its execution can be paused for a significant length of time at any point, due to reasons like:
            * a garbage collector (GC) that occasionally needs to stop all running threads
            * a virtual machine that is suspended and resumed during live migration from one host to another without reboot
            * a laptop that is suspended when the user closes the lid
            * when the operating system context-switches to another thread, pausing the currently running thread
            * an application performing synchronous disk access, making the thread to pause and wait for a slow disk I/O
            * the operating system is configured to allow swapping to disk (paging), a simple memory access result in a page faultt that requires a page from disk to be loaded into memory.
            * a Unix process can be paused by sending it the SIGSTOP signal and resume with SIGCONT signal later
        * response time guarantees: developing real-time systems is very expensive (requires support from all levels of the software stack), and they are most commonly used in safety-critical embedded devices.
        * limiting the impact of garbage colletion:
            * treat GC pauses like brief planned outages of a node, and to let other nodes handle requests from clients while one node is collecting its garbage. (an emerging idea)
            * use the garbage collector only for short-lived objects (which are fast to collectt) and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC of long-lived objects.
# Knowledge, Truth, and Lies

    Reliable behavior is achievable, even if the underlying system model provides every few guarantees. We need to think about assumptions we can make and the guarantees we may want to provide in a distributed system.
    * The Truth is Defined by the Majority:
        
        A node cannot trust its own judgment of a situation. A distributed system cannot exclusively rely on a single node, instead, they usually rely on a **quorum** (voting among the nodes).
        * fencing tokens:
            * problem: a node continues acting as the chosen one (an abusive client), even though the majority of nodes have declared it dead, causing the system to do something incorrect. (eg. a node still writes despite lock lease expires)
            * solution: fencing - a monotonically increasing counter kept by lock service. writes with an older token than one that has already ben processed are rejected
    * Byzantine Faults

        A node deliberately lies and subverts the system's guarantees. A system is Byzantine fault-tolerant if it continues to operate correctly even if some of the nodes are mulfunctioning and not obeying the protocol, or if malicious attackers are interfering with the network (eg. aerospace environments; a system with multiple organizations). Protocols for making systems Byzantine fault-tolerant are quite complicated. Traditional mechanisms (authentication, access control, encryption, firewalls, etc.) are still the main protection against attackers than Byzantine fault protection.
    * System Model and Reality
        * goal: helpful for distilling down the complexity of real systems to a manageable set of faults that we can reason about, so that problems can be solved in a systematic way.
        * system models:
            * system models for timing:

                formalize the kinds of faults that are expected to happen in a system (abstracted from hardware and software configs)
                1. synchronous model: bounded network delay, process pauses, and clock error (not realistic)
                1. partially synchronious model: the system behaves like a synchronous sytem most of the time, but it sometimes exceeds the bounds for network delay, process pauses, and clock error. (a realistic model of many systems)
                1. asynchronous model: not any timing assumptions for network delay, process pauses, and clock error.
            * system models for node failures:
                1. crash-stop faults: a node fails and never comes back
                1. crash-recovery faults: a node fails and comes back after some unknown time (most common)
                1. byzantine faults: a node may do absolutely anything including deceiving other nodes

                most common combination: partially synchronous model with crash-recovery faults
        * correctness of an algorithm:

            to define what it means for an algorithm to be correct, we can describe its properties. An algorithm is correct in some system model if it always satisfies its properties in all situations that we assume may occur in that system model. Proving an algorithm correct doesn't mean its implementation will always behave correctly, but it's a good first step. Take fencing token for example, we may require the algorithm to have the following properties:
            1. Uniqueness: no two requests for a fencing token return the same value
            1. monotonic sequence
            1. availability  
            * safety and liveness
                * safety: nothing bad happens (eg. the uniqueness and monotonic sequence properties for the fence token algorithm)
                * liveness: something good eventually happens (eg. the availability property for the fence token algorithm)
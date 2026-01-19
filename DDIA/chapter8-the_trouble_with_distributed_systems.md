# Chapter 8 - The Trouble with Distributed Systems

---

## Anything that can go wrong will go wrong

Working with a distributed system is fundamentally different from writing software on a single computer. The task as engineers is to build systems that meet the guarantees that users are expecting, in spite of everything going wrong.

---

## Faults and Partial Failures

An individual computer with good software is usually either fully functional or entirely broken (due to hardware faults), but not something in between. However, in distributed systems, nondeterminism and the possibility of partial failures makes it hard to work with. We need to build a reliable system from unreliable components.

---

## Problems with Distributed Systems

Detecting faults is hard (usually uses timeouts but can't distinguish between network and node failures). Tolerating a fault is not easy either (the system shares nothing and relies on messages).

### Problems

Packets can be lost, reordered, duplicated, or arbitrarily delayed in the network; clocks are approximate at best; and nodes can pause (e.g., due to garbage collection) or crash at any time.

#### 1. Unreliable Networks

**Issue**: The internet and most internal networks in datacenters (often Ethernet) are asynchronous packet networks with unbounded delays (the network gives no guarantees as to when the packet will arrive, or whether it will arrive at all).

**Possible scenarios:**
- Request is lost
- Request is queued and will be delivered later
- The remote node fails
- The remote node temporarily stops responding
- The remote node's response has been lost
- The remote node's response has been delayed

**Solution**: Timeout (still don't know whether the request is processed or not).

**Network faults in practice**: All things can go wrong, your software needs to be able to recover from them (e.g., cluster could become deadlocked and permanently unable to serve requests, even when the network recovers, or it could delete all of your data).

**Detecting faults**: The uncertainty about the network makes it difficult to tell whether a node is working or not. If you want to be sure that a request was successful, you need a positive response from the application itself. In general, you have to assume that you will get no response at all. You can retry a few times (at the application level), wait for a timeout to elapse, and eventually declare the node dead if it times out.

**Timeouts and unbounded delays**: How long the timeout should be is not a simple question. The problems with low timeouts are: retry may end up processing the same action twice; additional load on the nodes and the network when transferring work to other nodes.

**Network congestion and queueing scenarios:**
- Packet is dropped when the network switch queue fills up
- A request is queued by the operating system until the CPU core is ready
- In virtualized environments, the request is queued by the virtual machine monitor and paused for a short time while another virtual machine uses a CPU core
- TCP has flow control of sending requests
- TCP implements timeouts and automatic retry

**Network congestion** is the worst when system is highly utilized, by yourself or your "noisy neighbor" who shares the resources with you.

**How to choose timeouts:**
- By experiments: Measure round-trip times over an extended period over many machines
- Automatic timeouts adjustment (e.g., **Phi Accrual failure detector**; TCP retransmission timeouts)

**TCP vs. UDP**: UDP is a good choice in situations where delayed data is worthless (e.g., videoconferencing) because it does not perform flow control and packet retransmission like TCP does.

**Synchronous vs. Asynchronous networks** (latency guarantee vs. resource utilization):

1. **Synchronous - circuit network**: A fixed amount of reserved bandwidth which nobody else can use
   - **Pros**: Bounded delay (more predictable)
   - **Cons**: Not optimized for bursty traffic (reduced utilization)

2. **Async - TCP**: The packets opportunistically use whatever network bandwidth is available
   - **Pros**: Optimized for bursty traffic
   - **Cons**: Unbounded delay

#### 2. Unreliable Clocks

**Clocks are used for:**

1. Measuring durations (e.g., timeouts, response time, throughput, how long did the user spend on the site)
2. Measuring points in time (e.g., when is the article published, what time to send the notification, when does the cache expire, what is the timestamp of an error message in the log file)

**Problem:**

1. In a distributed system, it's difficult to determine the order in which things happened since message need to travel across the network from one machine to another
2. Clocks on each machine are not synchronized

**Monotonic vs. Time-of-Day Clocks:**

1. **Time-of-day clocks**: Returns the current date and time according to some calendar; synchronized with the NTP server
2. **Monotonic clocks**: Measuring a duration, such as a timeout or a service's response time; guaranteed to always move forward

> **Note**: In a distributed system, using a monotonic clock for measuring elapsed time is usually better since it doesn't assume any synchronization between different nodes' clocks.

**Clock synchronization and accuracy** (for time-of-day clocks): Unfortunately, synchronization methods aren't that reliable and accurate:

- Clock drifts due to temperature change of the machine
- Clock refuses to synchronize when differs too much from an NTP server
- Clock seeing time going back when NTP resets it
- A node is accidentally firewalled off from NTP server
- NTP synchronization can only be as good as the network delay
- NTP goes wrong or misconfigured
- Leap seconds issue
- In virtual machines, the hardware clock is virtualized which is unreliable
- Users deliberately set their hardware clock to incorrect date and time to circumvent timing limitations

**Relying on synchronized clocks**: If you use software that requires synchronized clocks, it is essential that you also carefully monitor the clock offsets between all the machines. Here are scenarios where clocks can be misused:

- **Timestamps for ordering events**: When last-write-wins strategy is used to resolve conflicts, a write(B) that is causally later than another write(A) may ended up having an earlier timestamp due to clock synchronization issues. Better using **logical clocks** that tracks the relative ordering of events.
- **Clock readings have a confidence interval**: It's better to think of a clock reading as a range of times, within a confidence interval (e.g., a system is 95% confident that the time now is between 10.2 and 10.5 seconds past the minute); most systems don't expose this uncertainty except for **Google's TrueTime API** in **Spanner**.
- **Synchronized clocks for global snapshots**: Using confidence level reading to determine the order of transaction IDs among multiple nodes for snapshot isolation implementation (where a monotonically increasing transaction ID is used).

#### 3. Process Pauses

A node in a distributed system must assume that its execution can be paused for a significant length of time at any point, due to reasons like:

- A garbage collector (GC) that occasionally needs to stop all running threads
- A virtual machine that is suspended and resumed during live migration from one host to another without reboot
- A laptop that is suspended when the user closes the lid
- When the operating system context-switches to another thread, pausing the currently running thread
- An application performing synchronous disk access, making the thread to pause and wait for a slow disk I/O
- The operating system is configured to allow swapping to disk (paging), a simple memory access result in a page fault that requires a page from disk to be loaded into memory
- A Unix process can be paused by sending it the `SIGSTOP` signal and resume with `SIGCONT` signal later

**Response time guarantees**: Developing real-time systems is very expensive (requires support from all levels of the software stack), and they are most commonly used in safety-critical embedded devices.

**Limiting the impact of garbage collection:**

- Treat GC pauses like brief planned outages of a node, and to let other nodes handle requests from clients while one node is collecting its garbage. (an emerging idea)
- Use the garbage collector only for short-lived objects (which are fast to collect) and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC of long-lived objects.

---

## Knowledge, Truth, and Lies

Reliable behavior is achievable, even if the underlying system model provides very few guarantees. We need to think about assumptions we can make and the guarantees we may want to provide in a distributed system.

### The Truth is Defined by the Majority

A node cannot trust its own judgment of a situation. A distributed system cannot exclusively rely on a single node, instead, they usually rely on a **quorum** (voting among the nodes).

**Fencing tokens:**

- **Problem**: A node continues acting as the chosen one (an abusive client), even though the majority of nodes have declared it dead, causing the system to do something incorrect. (e.g., a node still writes despite lock lease expires)
- **Solution**: **Fencing** - a monotonically increasing counter kept by lock service. Writes with an older token than one that has already been processed are rejected.

### Byzantine Faults

A node deliberately lies and subverts the system's guarantees. A system is **Byzantine fault-tolerant** if it continues to operate correctly even if some of the nodes are malfunctioning and not obeying the protocol, or if malicious attackers are interfering with the network (e.g., aerospace environments; a system with multiple organizations). 

Protocols for making systems Byzantine fault-tolerant are quite complicated. Traditional mechanisms (authentication, access control, encryption, firewalls, etc.) are still the main protection against attackers than Byzantine fault protection.

### System Model and Reality

**Goal**: Helpful for distilling down the complexity of real systems to a manageable set of faults that we can reason about, so that problems can be solved in a systematic way.

**System models:**

**System models for timing**: Formalize the kinds of faults that are expected to happen in a system (abstracted from hardware and software configs).

1. **Synchronous model**: Bounded network delay, process pauses, and clock error (not realistic)
2. **Partially synchronous model**: The system behaves like a synchronous system most of the time, but it sometimes exceeds the bounds for network delay, process pauses, and clock error. (a realistic model of many systems)
3. **Asynchronous model**: Not any timing assumptions for network delay, process pauses, and clock error.

**System models for node failures:**

1. **Crash-stop faults**: A node fails and never comes back
2. **Crash-recovery faults**: A node fails and comes back after some unknown time (most common)
3. **Byzantine faults**: A node may do absolutely anything including deceiving other nodes

> **Note**: Most common combination: partially synchronous model with crash-recovery faults.

**Correctness of an algorithm**: To define what it means for an algorithm to be correct, we can describe its properties. An algorithm is correct in some system model if it always satisfies its properties in all situations that we assume may occur in that system model. Proving an algorithm correct doesn't mean its implementation will always behave correctly, but it's a good first step. 

Take fencing token for example, we may require the algorithm to have the following properties:

1. **Uniqueness**: No two requests for a fencing token return the same value
2. **Monotonic sequence**
3. **Availability**

**Safety and liveness:**

- **Safety**: Nothing bad happens (e.g., the uniqueness and monotonic sequence properties for the fence token algorithm)
- **Liveness**: Something good eventually happens (e.g., the availability property for the fence token algorithm)

---

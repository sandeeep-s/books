# Problem with Distributed Systems
As we come to understand various edge cases that can occur in real systems, we get better at handling them.

Working with distributed systems is fundamentally different from writing software on a single computer--and the main difference is that there are lots of new and excitings ways in which things can go wrong.

In the end, our task as engineers is to build systems that do their job(i.e meet the guarantees that users are expecting), in spite of everything going wrong.

**The consequences of issues such as *unreliable networks*, and *unrialiable clocks* can be disorienting. So we need to know how to think about the state of a distributed system and how to reason about the things that may have happened.**

## Faults and Partial Failures
An individual computer with good software is usually either fully functional or entirely broken, but not something in between. There is no fundamental reason why software on a single node should be flaky
- When the hardware is working correctly, the same opration produces the same result(its determenistic).
- If there is a hardware problem, the consequence is usually a total system failure.

Always-correct computation is a deliberate choice in design of compouters:
- If an internal fault occurs, we prefer for computer to crash completely rather than returning a wrong result, because wrong results are difficult and confusing to deal with.
- Thus, the computers hide the fuzzy physical reality on which they are implemented and present an idealized system model that operates with mathematical precision.
    - A CPU instruction always does the same thing
    - If you write some data to memory or disk, that data remains intact and doesn't get randomly corrupted.

In distributed systems the situation is fundamentally different
- We are writing software that runs on several computers, connected by a network. 
- We are no longer working in a n idealized system model.

**Partial Failure:**
- In a distributed system, there may well be some parts of the system that are broken in some unpredicatble way, even though other parts of the system may be working fine. This is called *partial failure*.
- The difficulty is that partial failures are non-deterministic: 
    - If we try to do anything involving multiple nodes and the network, it may sometimes work and sometimes unpredictabley fail.
    We may not even know whether something succeeded or not, as the time it takes for a message to travel across a network is also non-deterministic.
- This nondetrminism and partial failures is what makes distributed systems hard to work with.
### Cloud Computing and Supercomputers
Supercomputer is more like a single-node system than a distributed system: it deals with partial failure by letting it escalate into total failure--id any part of the system fails, just let everything crash.

We focus on systems for implementing internet services, which usually look very different from supercomputers.
- Many internet-related applications are *online*.
    - They need to able to serve users with low latency at any time. 
    - Making the service unavailable is not an option
- Nodes in cloud services are built from commodity machines
    - They can provide good performance at lower cost due to economies of scale
    - But they also have higher failure rates
- Large datacenter networks are often based on IP and Ethernet, arranged in Clos topologies to provide high bisection bandwidth.
- The bigger a system gets, the more likely it is that one of its components is broken.
    - Over time, broken things get fixed, and new things break.
    - In a system of thousands of nodes it is reasonable to asume that something is always broken.
    - When the error handling strategy consists of simply giving up, a large system can end up spending most of its time recovering from faults rather than doing useful work.
- If a system can tolerate failed nodes and still keep working as a whole, that is a very useful feature for maintenance and operations e.g. It allows performing rolling upgrades or replacing failed virtual machines without downtime.
- Ina geographically distributed environment(keeping data geographically close to your users to reduce latency), communication most likely goes over the internet, which is slow and unreliable compared to local networks.

If we want to make distributed systems work, we must accept the possibility of partial failures and build fault-tolerance mechanisms into the software. 
- We need to build a relible system from unreliable components.
- The fault handling must be part of software design and you(as an operator) need to know what behavior to expect from the software in case of a fault.
- It is important to consider a wide range of possible faults--even fairly unlikely ones--and to artificially create such situations in our testing environment.

In distributed systems, suspicion, pesimism, and paranoia pay off.
### Building A Reliable System From Unreliable Components
It is an old idea in computing to contruct a more reliable system from a less reliable underlying base.
For example:
- Error-correcting codes in digital transmission
- TCP providing more reliable transport layer over the IP which is unreliable.

Although the system can be more reliable than its underlying parts, there is always a limit to how much more reliable it can be. But even though the more reliable higher-level system is not perfect, it is still useful because it takes care of some of the tricky low-level faults, and so the remaining faults are usually easier to reason about and deal with.

## Unreliable Networks
**Shared Noting Distributed Systems**
- Distributed systems with *shared nothing architecture* are a bunch of machines connected by a network.
- The network is the only way those machines can communicate--we assume that each machine has its own memory and disk, and one machine cannot access another machines memory or disk(except by making a request to a service over the network).

Shared-nothing has become a dominant approach for building internet services due to various reasons:
- It's comparatively checp, because it requires no special hardware--it can make use of commoditized cloud computing services
- It can achieve high reliability through redundancy across multiple geographically distributed datacenters

**Asynchronous packet networks**
- The internet and most internal networks in datacenters(often Ethernet) are *asynchronous packet networks*. 
- In this kind of network, one node can send a message(a packet) to another node, but the network gives no guarantees as to when it will arrive, or whether it will arrive at all.
- If we send a request and expect a response, many things could go wrong.
    1. Your request may have been lost
    2. Your request may be waiting in a queue and will be delivered later(perhaps the network or recipient is overloaded)
    3. The remote node may have failed(perhaps it crashed or was powered down)
    4. The remote node may have temporarily stopped responding(perhaps it is experiencing a long GC pause), but will start responding again later.
    5. The remote node may have processed your request, but response has been lost over the network
    6. The remote node may have processed your request, but the response is delayed and will be delivered later(perhaps the network or your own machine is overloaded)
- The sender can't even tell if the package was delivered: the only option is for the recipient is to send a response message, which may in turn be lost or delayed.
- These issues are indistinguishable in an asynchronous network: the only information we have is that we haven't received the response yet. If you send a request to another node, and don't receive a response, it is *impossible* to tell why.
- The usual way of handling this issue is a *timeout*. However, when a timeout occurs, we still don't know if the remote node received your request or not.
### Network Faults in Practice
Nobody is immune to network problems. Whenever any communication happens over a network it may fail--there is no way around it.

**Network Partitions:** When one part of the network is cut off from the rest, it is called a *network partition* or *netsplit*.

If the error handling of network faults is not defined and tested, arbitrarily bad things could happen. If software is put in unanticipated situation, it may do arbotrary unexpected things.

Handling network faults doesn't mean tolerating them. However, we do need to know how our software reacts to network problems and encure that the system can recover from them. 
### Detecting Faults
Many systems need to automatically detect faulty nodes. Unfortunately the uncertainty about the network makes it very difficult to tell whether a node is working or not. 

In specific circumstances we might get feedback to explicitly tell us that something is not working. Rapid feedback about a remote node being down is useful but we can't count on it. Even if TCP acknowledged that the packet was delivered, tha application may have crashed before handling it.

*If we want to be sure that a request was successful, we need a positive response from the application itself.*

Conversely, if something has gone wrong, we may get an error response at some level of the stack, but in general we have to assume that we will get no response at all. We can retry a few times(TCP retries transaparently, but we can retry at application level), wait for a timeout to elapse, and eventually declare the node dead if we don't hear back within the timeout.
### Timeouts and Unbounded Delays
Timeout is the only sure way of detecting faults. But there is no simple answer to how long the timeout should be.
- A long timeout means a long time until a node is declared dead(and during this time, users may have to wait or see error messages)
- A short timeout detects fault faster, but carries the risk of incorrectly declaring the node dead when in fact it has only suffered a temporary slowdown(e.g. due to a load spike on the node or network)
    - If another node takes over in such scenario, action might end up being performed twice
    - If a node was just slow to respond due to overload; transferring its load to other nodes may cause a cascading failure.

Asynchronous systems have *unbounded delays* and most server implementations cannot guarantee that they can handle requests within some maximum time. For failure detection, it's not sufficient for the system to be fast most of the time:if our timeout is low, it only takes a transient spike in round-trip times to throw the system off-balance.
#### Network congestion and queuing
*The variability of packet delays in computer networks is mainly due to queuing.*
- *Network Congestion:* 
    - If several different nodes simultaneously try to send packets to the same destination, the network switch may queue them up and feed them into the destination network link one by one. 
    - On a busy network link, a packet may have to wait a while before it gets a slot(this is called *network congestion*).
    - If there is so much incoming data, that the switch queue fills up, the packet is dropped, so it needs to be resent--even though the network is functioning fine.
- Whena a packet reaches the destination machine, if all the CPU cores are busy, the network packet is queued by the operating system until the application is ready to handle it. Depending on load on the machine, this may take up an arbitrary length of time.
- In virtualized environments, a running operating system is often paused for tens of milliseconds while another virtual machine uses a CPU core. During this time, the VM cannot consume any data from the network, so the incoming data is queued by the VM monitor, further increasing the variability of network delays.
- TCP performs flow control(aka *congestion avoidance* or *backpressure*), in which a node limits its own rate of sending in order to avoid overloading a network link or the receiving node.This means additional queuing at the sender even before th data enters the network.
- TCP considers a packet to be lost if it is not acknowledged within some timeout(which is calculated based on observed round-trip times), and lost packets are automatically retransmitted. Application sees the resulting delay.
- Queuing delays have an especially wide range when a system is close to it's maximum capacity. 

**TCP vs UDP**
Some latency sensitive applications, such as videoconferencing and VoIP, use UDP rather than TCP. It's a tradeoff between reliability and variability of delays.
- As UDP does not perform flow control and does not retransmit lost packages, it avoids some of the reasons for variable network delays.
- However, it is still susceptible to switch queues and scheduling delays.
- *UDP is a good choice in situations where delayed data is worthless.*

*Noisy neighbour*: 
- In public clouds and multi-tenant datacenters, resources are shared among many customers: the network links and switches, and even each machine's network link and CPUs are shared.
- Network delays can be highly variable if some near you is using a lot of resources(e.g. running batch workloads like MapReduce)
- In such environments, we can only choose timeouts experimentally using observed response time distribution. Even better, systems can continually measure response times(jitter), and automatically adjust timeouts.
#### Synchronous vs Asynchronous Networks
Comparing datacenter networks to traditional fixed-line telephone networks:

*Telephone networks:*
- A phone call requires a constantly low end-to-end latency and enough bandwidth to transfer the audio samples of our voice.
- A fixed line telephone network is extremely reliable: delayed audio frames and dropped calls are extremely rare.
- Telephone network is a circuit-switched network
    - When we make a call over the telephone network, it establishes a *circuit*: 
    - A fixed, guaranteed amount of bandwidth is allocated for the call, over the entire route between the two callers. - The circuit remains in place until the call ends.
- This kind of network is synchronous:even as data passes through several routers, it doesn't suffer from queuing.
- *Bounded delay:* Because there is no delay, the maximum end-to-end latency of the network is fixed. 

*Datacenter networks:*
- Datacenter networks and internet are optimized for *bursty traffic*. Requesting a web page, sending an email, or transferring a file doesn't have any specific bandwidth requirement--we just want it to complete as quickly as possible.
- Using circuits for bursty data traffic wastes network capacity, and makes transfers unnecessarily slow.
- By contrast, TCP dynamically adapts the rate of data transfer to the available network capacity.
- Ethernet and IP are packet-switched protocols, which suffer from queuing and thus unbounded delays in the network
- With careful use of *quality of service*(QoS, prioritization and scheduling of packets) and *admission control*(rate-limiting senders), it is possible to emulate circuoit switching on packet networks, or provide statistically bounded delay.

Currently deployed technology does not allow us to make any gurantees of the reliability of the network: we have to assume that the network congestion, queuing, and unbounded delays will happen.

Consequently there is no correct value for timeouts--they need to be determined experimentally.

**Latency and Resource Utilization**
More generally we can think of variable delays as a consequence of dynamic resource partitioning.
- Internet shares network bandwidth *dynamically*.
- Network switches decide which packet to send(i.e. bandwidth allocation) from one moment to next.
- This approach has downside of queuing, but the advantage is that it maximizes the utilization of the wire. the wire has a fixed cost, so if we utilize it better, each byte we send over the wire is cheaper.

Latency guarantees are possible in certain environments, if resources are statically partitioned(e.g. dedicated hardware and exclusive bandwidth allocation). However it comes at the cost of reduce utilization--in other words it is more expensive.

Variable delays in network are not a law of nature, but simply the result of a cost/benefit trade-off.

## Unreliable Clocks
Applications depned on clocks to answer questions about *durations* and *points in time*.

In a distributed system, it takes time for information to travel overacross the network from one machine to another. The delays are unbounded and variable. This fact makes it sometimes difficult to identify the order in which things happened if multiple machines are involved.

Each machine on the network has its own clock device, which is an actual hardware device: usually a quartz crystal oscillator. 
- These devices are not perfectly accurate, so each machine has its own notion of time, which may be slightly fater or slower than on other machines.
- *Clock Synchronization:* 
    - It is possible to synchronize clocks to some degree
    - Network Time Protocol allows computer clock to be adjusted according to the time reported by group of servers.
    - The servers in turn get their time from a more accurte toime source, such as a GPS receiver.
### Monotonic vs Time-of-Day clocks
Modern computers have at least two different kinds of clocks:
- A time-of-day clock
- A monotonic clock
#### Time-of-day clocks
A time-of-day clock returns current date and time according to some claendar(aka *wall-clock-time*).
- Time-of-day clocks are usually synchronized with NTP, which means that timestamp from one machine (ideally) means the same as a timestamp from another machine.
- Time-of-day clocks also have some oddities.
    - If the local clock is too far ahead of the NTP server, it might be forcibly reset and appear to jump back to a previous point in time. These and the jumps caused by leap seconds make time-of-day clocks unsuitable for measuring elapsed time.
    - Time-of-day clcoks historically had quite a coarse-grained resolution
#### Monotonic clocks
A monotonic closk is suitable for measuring duration(time interval). 
- They are guaranteed to move forward in time.
- The absolute value of the clock is meaningless. It makes no sense to compare monotonic clock values from two computers.
- On aserver with multiple CPU sockets, there may be seperate time per CPU, which is not necessarily synchronized with other CPUs
- NTP may adjust the frequency at which monotonic clocks move forward(*slewing the clock*) if it detects that the computer's local quartz is moving faster or slower than the NTP server.
- The resolution of monotonic clocks is usually quite good: on most systems they can measure time intervals in microseconds or less.

Ina distributed system, using monotonic clock for measuring elapsed time is usually fine, because it doesn't assume any synchronization between different node's clocks and is not sensitive to slight inaccuracies of measurement.
### Clock Synchronization and Accuracy
Our methods for getting the clock to tell the correct time aren't accurate.
- The quartz clock in a computer isn't very accurate. 
    - It *drifts*(runs faster or slower than it should). 
    - Clock drift varies according to the temperature of the machine. 
    - This drift limits the mest possible accuracy we can achieve, even if everything is working correctly.
- If a computer's clock differs too much from the NTP server, it may refuse to synchronize, or the local clock will be forcibly reset. Any applications observedrving the time before and after this reset may see time go backward or suddenly jump forward.
- If a node is accidently firewalled from NTP servers, the misconfiguration may go unnoticed for some time.
- NTP synchronization can only be as good as the network delay, so there is limit to it's accuracy when we are on a congested network with variable packet delays.
- Some NTP servers are wrong or  misconfigured, reporting time that is off by some hours. NTP clients are quite robust, because they query several servers and ignore outliers.
- Leap seconds result in a minute that is 59 seconds or 61 seconds long, which meses up timing assumptions on systems that are not designed with leap seconds in mind.
- In virtual machines hardware clock is virtualized.
    - Whe a CPU core is shared betwee virtual machines, each VM is paused for tens of milliseconds while another vM is running.
    - From an application's point of view, this pause manifests itself as the clock suddenly jumping forward.
- If we run software on devices that we don't fully control(e.g. mobile or embedded devices), you cannot probably trust the device's ahardware clock at all.

It is possible to achieve very good clock accuracy by using GPS receivers, the Precision Time Protocol(PTP), and careful deployment and monitoring.
### Relying on Synchronous Clocks
The problems with the clocks is that while they seem simple and easy to use, they have a surprising number of pitfalls. Although they work quite well most of the time, robust software needs to be prepared to deal with incorrect clocks.

Part of the problem is that incorrect clocks easily go unnoticed. If we use software that requires synchronized clocks, it is essential that we carefully monitor the clock offsets between all the machines. Any node whose clock drifts too far from others should be declared dead and removed from the cluster. Such monitoring ensures that we notice broken clocks before they can cause too much damage.
#### Timestamps for ordering events
It is dangerous to rely on clocks for *ordering of events across multiple machines*.

The *last write wins* conflict resolution strategy can be dangerous when using timestamps for event ordering.
- Database writes can mysteriously disappear
- LWW cannot distinguish between the writes that occurred sequentially in quick succession, and writes that were truly concurrent.
- it is possible for two nodes to independently generate writes with the same timestamp, especially when clock has only millisecond resolution.

*Logical clocks*, which are based on incrementing counters rather than an oscillating quartz crystal, are a safre alternative for ordering events. Logical clocks measure only the relative ordering of events.
#### Clock readings have a confidence interval
It doesn't make sense to think of a clock reading as point in time--it is more like a range of times within a confidence interval: for example a system may be 95% confident that the time is now beteen 10.3 and 10.5 seconds after the last minute.

The uncertainty bound can be calculated based on our time source.
- If we have a GPS receiver or atomic(caesium) clock directly attached to our computer, the expected error range is reported by the manufacturer.
- If we are getting time from a server the uncertainty is based on:
    - The expected quartz drift since our last sync with eth server
    - Plus the NTP server's uncertainty
    - Plus th network round-trip time to the server.

Google's TrueTime API in Spanner, explicitly reports the confidence interval on the local clock. When we ask it for the current time, we get back two values:[*earliest, latest*] which are the earliest possible and latest possible timestamp.
#### Synchronized clocks for global snapshots
The most common implementation of snapshot isolation requires monotonically increasing transaction ID.

On a single node database, a simple counter is sufficient for generating transaction IDs.

However, in a distributed database, a global monotonically increasing transaction ID is difficult to generate, because it requires coordination.
    - The transaction ID must reflect causality.
    -With lots of small, rapid transactions creating transaction IDs in a distributed database becomes an untenable bottleneck.

Spanner implements snapshot isolation across datacenters using timestamps as transaction IDs. 
-It uses the confidence interval reported by the Google's Truetime API and is based on the following observation. 
    - If two confidence intervals A=[Aearliest, Alatest] and B=[Bearliest and Blatest] and those two intervals do not overlap, (i.e. Aearliest < Alatest < Bearliest < Blatest), then B definitely happened after A.
    - Only if intervals overlap are we unsure in which order A and B happened.
- In order to ensure transaction timestamps reflect causality, Spanner deliberately waits for the length of the confidence interval before committing a read-write transaction. By doing this, it ensures thatany transaction that may read the data is at a sufficiently later time, so that their timestamps do not overlap. In order to keep the wait time as short as possible, Spanner needs to keep clock uncertainty as small as possible;for this purpose Google deploys a GPS receiver or atomic clock in each datacenter.
#### Process Pauses
There are many reasons why a thread might be paused in the middle of procesing.
- The *stop-the-world GC* pauses
- The VMs can be *suspended* and *resumed* e.g. for live migration of VMs from one host to another
- On end-user devices like laptops, execution may be suspended and resumed arbitrarily e.g. when the user closes lid of the laptop
- When the OS context-switches to another thread, or when the hypervisor switches to a different VM.
- If the application performs synchronous disk access, a thread may be paused waiting for a slow disk I/O operation to complete.
    - I/O pauses and GC pauses may even conspire to combine their delays.
    - If the disk is actually a network files system or a network block device, the I/O latency is further subject to variable network delays.
- If the OS is configured to allow *paging*, it may spend most of its time *thrashing*.
- A Unix process can be paused by sending it a SIGSTOP signal.

All of these occurences can *prempt* the running thread at any time and resume it at some later time, without the thread even noticing. The problem is similar to multi-threaded programming on a single node.]

For making multi-threaded code on single node thread-safe, we have tools like mutexes, semaphores, atomic counters, lock-free data-structures, blocking queues and so on. Unfortunately these tools don't directly translate to distributed systems because *a distributed system has no shared memory--only messages sent over an unreliable network*.

A node in a distributed system must assume that it's execution can be stopped for significant length of time at any point, even in the middle of a function. During the pause, the rest of the world keeps moving and may even declare the paused node dead because its not moving.
#### Response time guarantees
**Hard real time systems:** Some software runs in environments where a failure to respond within certain time can cause serious damage. In these systems, there is a specified dealine by which the software must respond; if it doesn't meet the deadline, that may cause a failure of the entire system. These are caled hard real-tiome systems.

In embedded systems, a real time means that the system is carefully designed and tested to meet specified training guarantees in all circumstances.

Providing real-time guarantees in a system requires support from all levels of the software stack:
- A *real time operating system*(RTOS) that allows processes to be scheduled with a guaranteed allocation of CPU time in specified intervals is needed.
- Library functions must document their worst case execution times
- Dynamic memory allocation may be restricted or disallowed entirely(real-time garbage collectors exist, but application must still ensure that it doesn't give GC too much work to do)

All of this requires a lot of additional work and severely restricts the range of programming langiuages, libraries, and tools that can be used(since most langiuages and tools do not provide real-time guarantees).
For these reasons, developing real-time systems is very expensive, and they are most commonly used in safety critical embedded devices.

Morever, real-time systems may have lower throughput, since they have to prioritize timely responses above all else.

For most server-side data processing systems, real-time guarantees are simply not economical or apporpriate.
#### Limiting the impact of garbage collection
The negative impacts of process pauses can be mitigated without resorting to expensive real-time scheduling guarantees. Language runtimes have some flexibility around when they can schedule garbage collections, because they can track teh rate of object allocation and the remaining free memory over time.

An emerging idea is to treat GC pauses like brief planned outages of node, and to let other nodes handle requests from clients while one node is colecting it's garbage.

Another idea is to use garbage collector only for short-lived objects(which are fast to collect), and to restart processes periodically, before they accumulate enough long-lived objects to require a full GC.

## Knowledge, Truth, and Lies
Distributed systems are diffeent than programs running on a single computers in following ways:
- There is no shared memory, only message passing via an unreliable network with variable delays
- The systems may suffer from partial failures, unreliable clocks, and processing pauses.

Understanding distributed system can be very disorienting
- A node in the network cannot know anything for sure--it can only make guesses based on the messages it receives(or doesn't receive) via the network.
- A node can only find out what state another node is in(what data it has stored, whether it is functioning correctly) by exchanging messages with it.
- If the remote node doesn't respond, there is no way of knowing what state it is in, because problems in network cannot be reliably distinguished from problems in node.

Discussions of these systems border on the philosophical
- What do we know to be true or false in our system?
- How sure can we be of that knowledge, if the mechanisms for perception and measurement are unreliable.
- Shoudl software systems obey the laws we expect of the physical world, such as cause and effect?

In a distributed system, we can state the assumptions we are making about the behavior(*the system model*) and design the actual system in such a way that it meets those assumptions. Algorithms can be proved to function correctly within a certain system model. This means that reliable behavior is achievable, even if the underlying system model provides very few guarantees.
### The Truth is Defined by the Majority
The distributed system cannot exclusively rely on a single node, because a node may fail at any time, potentially leaving the system stuck and unable to recover. Instead many distributed algorithms rely on a *quorum* that is voting among nodes: decisions require some minimum number of votes from several nodes in order to reduce dependence on any particular node.

Most commonly , the quorum is an absolute majority of more than half the nodes.A majority quorum allows the system to continue working if individual nodes have failed.(with three nodes, 2 failures can be tolearted, with 5 nodes, 3 failures can be tolerated). However, it is still safe because there can only be one majority in the system.
#### The leader and the lock
Frequently a system requires there to be only one of some thing.
- Only one leader is allowed to the leader of a partition, to avoid split brain
- Only one transaction is allowed to hold lock on a particukar resource or object  , to prevent concurrently writing to it and corrupting it.
- Only one user is allowed to register a particular username

Implementing this in a distributed system requires care: 
- Even if the node believes that it is the "chosen one", that doesn't necessarily mean a quorum of nodes agrees. 
- A node may have formerly been a leader, but if the other nodes declared it dead in the meantime, it may have been demoted and another leader may already have been elected. 
- If a node continues acting as the chosen one, , such a node could send other nodes messages in its self-appointed capacity. If other nodes believe it, system as whole could do something incorrect.
#### Fencing tokens
When using a lock or lease to protect some resource, we need to ensure that the node that is under false belief of being the chosen one cannot disrupt the rest of the system. We can use a simple technique called *fencing* to achive this.
- Every time the lock server grants a lock or lease, it also returns a *fencing token* which is a number that increases every time the access is granted.
- We can then require that every time a client sends a write request to the storage service, it must include it's current fencing token.

If Zookeeper is used as a lock service the transactionID zxID or the node version cversion can be used as fencing token. Since they are guaranteed to me monotonically increasing, they have the required properties.

This mechanism requires the resource itself to take active role in checking tokens by rejecting writes with an older token than one that has already been processed. Checking tokens on the server side is a good thing: it is unwise for a service to assume that it's clients will always be well-behaved because the clients are often run by people whose priorities are very different from the priorities of people running the server. thu it is a good ida for any service to prtect itself from accidently abusive clients.
#### Byzantine Faults
Distributed systems problems become much harder if there is a risk that nodes may "lie"(send arbitrary and corrupted responses). Such behavior is known as *Byzantine fault*, and the problem of reaching consensus in this untrusting environment is known as the *Byzantine Generals Problem*. 

A system id *Byzantine-fault tolerant* if it continues to operate correctly even if some of the nodes are malfunctioning and not obeying the potocol or if malicious attackers are interfering with the network.

For most server side data systems we can usually safely assume that there are no Byzantine faults.Protocols for making systems Byzantine-fault tolerant are quite complicated, fault tolerant embedded systems rely on support from the hardware level. For most server side data systems, the cost of deploying Byzantine fault-tolerant solutions makes them impracticable.

In peer-to-peer networks where there is no dentral authority, Byzantine fault tolerance is more relevant.

Web applications need to expect arbitrary and malicious behaviourof clients that are under end-user control, such as web browsers. We don't use Byzantine fault-tolerant protocols here, but simply make server the authority(input validation, sanitization , and output escaping) on deciding what client behavior is and is not allowed. 

Byzantine fault tolerant algoritham cannot save us from software bugs, vulnerabilities, security compromises, and malicious attacks. 
##### Weak forms of lying
Although we assume that nodes are generally honest, it can be worth adding mechanisms to software that guard against weak forms of "lying". For example: Invalid messages due to hardware issues, software bugs, and misconfiguration.

Such protection mechanisms are not full-blown Byzantine fault tolerance, as they would not withstand a determined adversary, but they are nevertheless simpel and pragmatic steps toward better reliability. For example:
- Network packets can get corrupted due to hardware issues, bugs in OS, drivers, routers etc. Simple measures like checksums in application-level protocol can be sufficient here
- A publicly accessible application must carefully sanitize any input from the users.
- NTP clients can be configured with multiple server addresses. When synchronizing, client contacts all of them, estimates their errors, and checks that majority of servers agree on some time range.
### System Model and reality
Distributed system algorithms need to tolerate varioud faults of distributed systems. 

Algorithms should not depend too heavily on the details of the hardware and software configuration. this in turn requires that we formalize the kind of faults that we expect to happen in teh system. We do this by defining a *system model*.

*System model is an abstraction that describes what things an algorithm may assume.*

With regards to timing assumptions, three system models are in common use:
- Synchronous model
    - The synchronous model assumes bounded network delay, bounded process pauses, and bounded clock error i.e network delay, pauses, and cloud drift will never exceed some fixed upper bound.
    - The synchronous model is not a realistic model of most practical systems, because unbounded delays and pauses do occur.
- Partially synchronous model
    - Partial synchrony means that a system behaves like a synchronous system *most of the time*, but it sometimes exceeds the bounds for network delays, process pauses and clock drift.
    - This is a realistic model of many systems:
        - Most of the time, networks and processs are quite well-behaved--otherwise we would never be able to get anything done.
        - But we have to reckon with thefact that any timing assumptions may be shattered occassionally.
- Asynchronous model
    - In this model, algorithm is not allowed to make any assumtions--infact it doesn't even have a clock(so it acnnot use timeouts).
    - Some algorithms are designed for the asynchronous model.

The three most common models for node failures are:
- Crash-stop faults
    - In the carsh-stop model, an algoruthm may assume that a node can fail in only one way, namely by crashing.
    - This means that the node may suddenly stop responding at any moment, and thereafter that node is gone forever--it never comes back.
- Crash-recovery faults
    - We assume that nodes may crash at any moment, and perhaps start responding againafter some unknown time.
    - Nodes are assumed to have stable storage(i.e. non-volatile disk storage) that is preserved across crashes, while the in-memory state is assumed to be lost.
- Byzantine(arbitrary) faults
    - Nodes may absolutely do anything, including trying to trick and decieve other nodes.

*For modeling real systems partiall synchronous model with crash-recovery faults is genrally the most useful model.*
#### Correctness of an algorithm
To define what it means for a distributed algorithm to be *correct*, we can describe its *properties*. for example, if we are generating fencing tokens for a lock, we may require the algorithm to have following properties.
- *Uniqueness:* No two requests for a fencing token return the same value
- *Monotonic Sequence:* If request x and y returned tokens tx and ty respectively, and x completed before y began, then tx < ty.
- *Availability:* A node that requires a fencing token and does not crash eventually receives a response.

An algorithm is correct in some system model if it always satisfies it's properties in all situations that we assume may occur in that system model.
#### Safety and Liveness
Algorithms have two kinds of properties:
- *Safety properties:* 
    - Informally defined as *nothing bad happens*.
    - If a safety property is violated, we can point at a particular point in time at which it was broken.
    - After a safety property is violated, the violation cannot be undone--the damage is already done.
- *Liveness properties:*
    - Informally defined as, *something good eventually happens*.
    - A liveness property may not hold at some point in time, but there is always hope that it may be satisfied in the future.

Distinguishing between safety and liveness properties helps us deal with difficult system models.
- For distributed algorithms, it is common to expect that safety properties always hold, in all possible situations of a system model. That is even if all nodes crash, or the entire network fails, the algorithm must nevertheless ensure that it doesn't return a wrong result.
- However, with liveness properties we are allowed to make caveats
    - For example, we could say that a request needs to receive a response only if a majority of nodes have not crashed, and only if the network eventually recovers from an outage.
    - the definition of partially synchronous model requires that eventually the system returns to a synchronized state.
#### Mapping system models to the read workloads
Safety and liveness properties and system models are very useful for reasoning about the correctness of a distributed algorithm. However when implementing an algorithm in practice, the messy facts of reality come back to bite us again, and it becomes clear that the system model is a simplified abstraction of reality.

The theoretical description of an algorithm can declare that certain things are simply not assumed to happen. However, a real implementation may still haveto include the code to handle the case where somethig happens that was assumed to be impossible.(This is arguably the difference between computer science and software engineering).

Theoretical, abstract system models are incredibly helpful for distilling down the complexity of real systems to amanageable set of faults that we can reason about, so that we understand the problem and try to solve it systematically.

*Theoretical analysis and empirical testing are both important*
- We can prove an algorithm correct by showing that their properties always hold in some system model. 
- Proving an algorithm correct does not mean it's implementation on areal system will always behave correctly. 
- But it's a very good step, because the theoretical analysis can uncover problems in a an an algorithm that might remain hidden for a long time ina real system, and that only come to bite us when our assumtions are defeated due to unusual circumstances.



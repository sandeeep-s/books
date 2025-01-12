# Reliable, Scalable and Maintainable Apps
*The internet was done so well that most people think of it as a natural resource like the Pacific Ocean, rather than something that was manmade. When was the last time a technology with a scale like that was so error free.*

*Standard Building Blocks:* A data-intensive application is typically built using standard building blocks that provide commonly needed functionality. Many applications need to:
- **Store data** so that they, or another application, can find it again later(*databases*)
- **Remember the result of an expensive operation**, to speed up reads*(*caches*)
- Allow users to **search data by keywords** or **filter it in various ways**(*search indexes*)
- **Send a message to another process**, to be handled **asynchronously**(*stream processing*)
- **Periodically crunch** a large amount of **accumulated data**(*batch processing*)

*Choose Right Tool for the Job:*
- There are many implementations with different characteristics for each of these building blocks, because different applications have different requirements. 
- When building an application, we still need to figure out which tools and approaches are best for the task at hand. 
- We need to explore what different tools have in common, what distinguishes them, and how they achieve their characteristics.

*And it can be hard to combine tools when a single tool cannot accomplish the job.*

## Thinking About Data Systems
We typically think of databases, queues, caches etc. as being very different categories of tools. So why should we lump the all together under an umbrella term like data systems.
- Many new tools for data storage and processing are optimized for a variety of different use cases, an they no longer fit neatly into traditional categories
- Increasingly many applications now have wuch demanding or wide-ranging requirements.A single tool can no longer meet all of their data processing and storage needs. Instead, work is broken down into tasks that can be performed efficiently on a single tool, and those different tools are stiched together using application code.
- ![Figure 1-1]()

When you combine several tools in order to provide a service, the service's API hides the implementation details from the clients of the service, **essentially creating a new, special-purpose data system from smaller, general-purpose components**. Some factors that may influence design of such a data system are:
- Skills and experience of the people involved
- Legacy system dependencies
- Timescale for delivery
- Organization's tolerance of different kinds of risk
- Regulatory constraints

Top three design concerns that are important in most software systems
### - Reliability
The system should continue to work *correctly*(performing the correct function at the desired level of performance) even in the face of *adversity*(hardware or software faults, and human error).
#### Thinking About Reliability
For software to be reliable typically means:
- The applications performs the function users expected
- It's performance is good enough for the required use case, under the expected load.
- It can tolerate the user making mistakes, or using the application in unexpected ways
- The system prevents any unauthorized access and abuse
##### Fault-tolerant(or Resilient) Systems:
###### Faults
**Faults** are the things that can go wrong. It is usually defined as one component of a system deviating from it's spec. Fault is not the same as a failure. 

**Failure** is when the system as a whole stops providing the required service to the user.

*Systems that can anticipate faults and cope with them are called fault-tolerant or resilient.*

In general we prefer tolerating faults over preventing faults.
- It is not possible to make a system  tolerant of every kind of fault. 
- It is impossible to reduce the probability of a fault to zero.
- Therefore it is usually best to design fault-tolerance mechanisms that prevent faults from causing failures. 
###### Hardware Faults
Hard disks crash, RAM becomes faulty, the power grid has a blackout, somebody unplugs a wrong network cable.

*Tolerating loss of individual hardware components*
- For hardware fault-tolerancy, our first response is usually to add redundancy to the individual hardware components in order to reduce the failure rate of the system. 
- When one component dies, redundant component can take its place while the broken component is replaced.

*Tolerating loss of entire machine*
- Until recently, redundancy hardware components was enough for most applications, since it makes total failure of a single machine fairly rare. 
- However, as data volumes have increased, more applications have begun using larger numbers of machines, which proportionately increases the rate of hardware faults. 
- Hence there is a move towards the systems that can tolerate the loss of entire machines, by using *software fault-tolerance techniques in preference or addition to hardware redundancy.*
- Also, the systems that can be tolerate machine failure can be patched one node at a time, without downtime of the entire system thus providing *operational advantage*.
###### Software Errors
We usually think of faults as being random and independent of each other. It is unlikely that a large no of hardware components will fail at the same time.

Another class of fault is **systematic errors** wihin the system. 
- Such faults are harder to anticipate. 
- Because they are correlated across nodes, they tend to cause many more system failures than uncorrelated faults.
- For Example:
	- A software bug that causes every instance of an application server to crash when given a particular bad input.
	- A runaway process that uses some shared resource
	- A service that a system depnds on slows down, becomes unresponsive, or starts returning corrupted responses.
	- Cascading failurs, where a fault in one component triggers a fault in another component, which in turn triggers further faults.
- The bugs that cause these kind of software faults often lie dormant for a long time until they are triggered by an unusual set of circumstances. In those circumstances it is revealed that the software is making some assumptions about the environment--and while that assumption is usually true, it stops being true for some reason.
- There is no quick solution to the problem of systematic faults in software. Lots of small things can help:
	- Carefully thinking about assumptions and interactions in ths system
	- Thorough testing
	- Process isolation
	- Allowing processes to crash and restart
	- Measuring, monitoring and analyzing system behavior in production
	- If a system is expected to provide some guarantee, it can constantly check itself while running and raise an alert if discrepancy is found.
###### Human Errors
Even when they have best intentions, humans are known to be unreliable. 

How do we make our systems reliable, in spite of unreliable humans?
- Design systems in a way that minimizes opportunity for error
- Decouple the places where people make the most mistakes
- Test thorougly at all levels, from unit tests to whole system-integration tests and manual tests.
- Allow quick and easy recovery from human errors, to minimize the impact in case of failures.
- Set up detailed and clear monitoring, such as performance metrics and error rates.
- Implement good management practices and training

### Scalability
Scalability is the term we use to describe a system's ability to cope with the increased load. As the system grows(in data volume, traffic volume, or complexity), there should be reasonable ways of dealing with that growth. 

Discussing scalability means considering questions like:
- If the system grows in a particular way, what are our options to cope with the growth?
- How can we add computing resources to handle the increased load?
#### Describing load
Load can be described by a few numbers which we call **load parameters**. 

The best choice of load parameters depends on the architecture of the syste. They may be:
- Requests per second to a web server
- The ratio of reads to writes in a database
- The number of simultaneously active users in a chat room
- The hit rate on a cache
#### Describing Performance
Once you have described the load on your system, you can investigate what happens when the load increases. You can look at it in two ways.
- When you increase the load parameter and keep the system resources unchanged, how is the performance of your system affected?
- When you increase the load parameter, how much do you need to increase the resources to keep the system performance unchanged?

In *batch processing* systems we usually care about *throughput*. In *online systems*, a service's *response time* is more important.

Latency and response time are not the same.
- **Response time** is what client sees
- **Service time** is the actual time it takes to process a request
- **Latency** is the duration that a request is waiting to be handled--during which it is latent,awaiting service.
##### Response time
We need to think of the response time not as a single number, but as a distribution of values that you can measure. Even in a scenario where we'd think all requests should take the same time, you get variation. Random aditional latency could be introduced by:
- A context switch to a background process
- The loss of a network packet and retransmission
- A garbage collection pause
- A page fault forcing a read from the disk
- Mechanical vibrations in a server rack
- Many other causes
###### Percentiles
*Mean* is not a very good metric if you want to know your "typical" response time, because it doesn't tell you how many users actualy experienced that delay. Usually it is better to use *percentiles*.

*Tail Latencies:*
- High percentiles(p95, p99, p999) of response time are known as *tail latencies*. 
- They are important because they directly affect user experience of the service. 
- Reducing response times at very high percentiles(p9999) is difficult because they are easily affected by the random events outside of your control, and the benefits are diminishing.

*Head-of-Line Blocking:*
- Queuing delays often account for a large part of the response time at high percentiles. 
- A server can process only a small number of requests in parallel(limited, for example, by its number of CPU cores)
- It only takes a small number of slow requests to hold up the processing of subsequent requests--an effect sometimes known as *head-of-line blocking*.

*Tail Latency Amplification:*
- High percentiles become especially important in backend services that are called multiple times as part of serving a single end-user request.
- Even if you make the calls in parallel, the end user request still has to wait for the slowest of the parallel calls to complete. 
- Even if only a small percentage of backend calls are slow, the chance of getting a slow request increases if an end-user request requires multiple backend calls.
- And so a higher proportion of end-user requests end up being slow(an effect known as *tail latency amplification*)
#### Approaches for Coping with Load
An architecture that is appropriate for one level of load is unlikely to cope with 10 times that load. If you are working on a fast-growing service, it is likely that you will have to rethink your architecture on every order of maginitude increase in load
##### Dichotomy between Scaling Up and Scaling Out
*Scaling Up (aka Vertical Scaling):* Moving to a more powerful machine.
- A system that runs on a single machine is often simpler.
- High-end machines can become exepensive

*Scaling Out (aka Horizontal Scaling):* Distributing load across multiple smaller machines(*shared-nothing architecture*) 
- But high-end machines can become exepensive, so very intensive workloads can't often avoid scaling out.
- Distributing stateless services acros multiple machines is fairly straightforward
- Taking stateful data systems from a single node to a distributed setup can intrroduce a lot of additional complexity.

*In reality, good architectures usually invove pragmatic mix of approaches.*
##### Elastic vs Manually Scaled
*Elastically scaled systems:*
- These systems can automatically add computing resources when they detect a load increase.
- Elastic system can be useful if the load is highly unpredictable.

*Manually scaled systems:*
- These systems are scaled manually, meaning a human analyzes the capacity and decides to add more machines to the system.
- Manually scaled systems are simpler and may have few operational surprises.
#### Magic Scaling Sauce
The architecture of systems that operate at high scale is usually highly specific to the application. There is no such thing as a generic, one-size-fits-all architecture. The problem may be:
- The volume of reads
- The volume of writes
- The volume of data to store
- The complexity of the data
- The response time requirements
- The access patterns
- or usually some misxture of all these plus many more issues

The architecture that scales well for a particular application is built around the assumptions of which operations will be common and which will be rare--the load parameters.

Scalable architectures are nevertheless built from general-purpose building blcoks arranged in familier patterns.

### Maintainability
Over time, many people will work on the system(engineering and operations, bothe maintaining curent behavior and adapting the system to new use cases), and they should all be able to work on it productively. Examples include:

Majority of the cost of software is not in it's initial development, but in it's ongoing maintenance.
- Investigating failures
- Fixing bugs
- Repaying technical debt
- Keeping its systems operational
- Adapting it to new platforms
- Modifying it for new use cases
- Adding new features

We can and should design softyware in such a way that hopefully it minimizes pain during maintenance.
#### Design principles for maintainability
##### Operability - Making life easy for operations
Make it easy for the operations teams to keep running the systems smoothly

*Good operations can often work around the limitations of bad(or incomplete) software, but good software cannot run reliably with bad operations.*

A good operations team is typically responsible for the following:
- Monitoring the health of the system and and quickly restoring service if it goes into a bad state.
- Tracking down the cause of problems, such as system failures or downgraded performance.
- Keeping software and platforms up-to-date, including security patches
- Keeping tabs on how different systems affect each other, ao that problematic change can be avoided before it causes damage.
- Anticipating future problems and solving them before they occur(e.g. capacity planning)
- Establishing good practices and tools for deployment, configuration management, and more.
- Performing complex maintenance tasks, such as moving an application from one platform to another.
- Maintaining the security of the system as configuration changes are made
- Defining processes that make operations preictable and help keep the production environments stable.
- Preserving the knowledge about the system, even as individual people come and go.

Good operability means making routine tasks easy, allowing the operations team to focus their efforts on high-value activities. Data systems can do various things to make routine tasks easy.
- Providing visibility into the runtime behavior and internals of the system, with good monitoring.
- Avoiding dependency on individual machines(Allowing machines to eb taken down for maintenance while the system as awhole continues to run uninterrupted)
- Providing good documentation and easy to use operational model("If I do X, Y will happen")
- Providing good default behavior, hut also giving administrators the freedom to override defaults when needed.
- Self-healing where appropriate, but also giving administrators manual control over the system when needed.
- Exhibiting predictable behavior, minimizing surprises.

##### Simplicity - Managing Complexity
Make it easy for new engineers to understand the system, by removing as much complexity from the system as possible

Complexity in code slows down everyone who needs to work on the system, further increasing the cost of maintenance. 

*Some of Possible Symptoms of Complexity:*
- Explosion of the state space
- Tight coupling of modules
- Tangled dependencies
- Inconsistent naming and terminology
- Hacks aimed at solving performance problems
- Special-casing to work around issues elsewhere and many more

When complexity makes maintenance hard, budgets and schedules are often overrun. In complex software there is also a greater risk of introducing bugs when making a change: when the system is harder for developers to understand and reason about, hidden assumptions, unintended consequences and unexpected interactions are easily overlooked. 

Making a system simpler doesn't mean reducing it's functionality, it can also mean reducing it's accidental complexity. Complexity is accidental if it is not inherent in the problem the software solves, but arises only from its implementation. 

*Abstraction:*
- One of the best tool we have for removing accidental complexity is *abstraction*. 
- However, finding good abstractions is very hard. 
- In the field of distributed systems, there are many good algorithms.
- But it is much less clear how we should be packaging them into abstractions that can help us keep the complexity of the system at manageable levels.
##### Evolvability (aka Extensibility, Modifiability, o Plasticity) - Making Change Easy
Make it easy for engineers to make changes to the system in the future, adapting it for unanticipated use cases as requirements change.

System's requirements are more often than not in constant flux:
- We learn new facts
- Previously unanticipated use cases emerge
- Business priorities change
- Users request new features
- New platforms replace old platforms
- Legal or regulatory requirements change
- Growth of the system forces architectural changes
- Others

Agile working patterns and techniques(e.g. TDD, Refactoring etc.) focus on a fairly small local scale(A couple opf source code files within the same application). We need to search for ways to increase agility on the *level of a larger data system, perhaps consisting of sewveral different applications or services with different characteristics*.

The ease with which you can modify a data system and adapt it to changing requirements is closely linked to its simplicity and its abstractions. Simple and easy-to-understand systems are usually easier to modify than the complex ones.

# Replication
Replication means keeping a copy of the same data on different machines that are connected via a network. Common reasons for data replication are scalability(increase read throughput), fault tolerance(increase availability), performance(reduce access latency). 

All of the difficulty in replication lies in handling *changes* to replicated data. Three popular algorithms for replicating changes between nodes are:
- Single-leader replication
- Multi-leader replication
- Leaderless replication
## Single-leader replication (aka active/passive or master/slave replication)
*Replica:* Each node that stores copy of the data is called a replica. All data needs to be on all replicas. Every write to the database needs to be processed by every replica.

Single-leader replication works as follows
1. One of the replica is designated as leader(aka master or primary). Clients send their write requests to the leader, which first writes the new data to it's local storage.
2. Other replicas are known as followers(aka read replicas, slaves, secondaries or hot standbys). Whenever the leader writes data to its local storage, it also sends the data change to all of its followers as part of a *replication log* or *change stream*.  Each follower takes the log from the leader and updates the local copy of the database, by applying the writes in the same order as they were processed on the leader.
3. Client can query either the leader or followers for data.
4. The followers are read-only from client's point of view.

Leader-based replication is used by
- Relational and Non-relational databases
- Distributed message brokers
- Network filesystems
- Replicated block devices
### Synchronous vs Asynchronous Replication
#### Synchronous Replication:
Leader waits until follower has confirmed it received the write before reporting success to the client.

The advantage of synchronous replication is that the follower is guaranteed to have an up-to-date copy of the data that is consistent with the leader.

The disadvantage is that if the synchronous follower doesn't respond, the write cannot be processed. The leader must block all writes and wait until the synchronous replica is available again.
#### Asynchronous Replication:
Leader sends the message, but doesn't wait for response from the follower.

Normally, replication is very fast(most databases apply changes to followers in less than a second). However, there is no guarantee of how long it might take. Followers might fall behind leader by several minutes or more;for example if a follower is recovering from a failure, if the system is operating near maximum capacity, or if there are network problems between the nodes.
#### Semi-Synchronous Replication
It is impracticable for all followers to be synchronous: any one node outage would cause the whole system to grind to a halt. In practice, if we enable synchronous replication on database, it usually means that one follower is synchronous and others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that you have an up-to-date copy of data on at least two nodes.
### Setting Up New Followers
New follower(setup to increase no. of replicas or to replace failed nodes) needs to get an accurate copy of leader's data.

**Setting up a new follower without downtime**
1. Take a consistent snapshot of leader's database
2. Copy the snapshot to the new follower node
3. The follower connects to the leader and requests all the data changes that have happened since the snapshot was taken.
4. Once the follower has *caught up*(processed the backlog of data changes since the snapshot), it can continue to process data changes from the leader as they happen. 
### Handling Node Outages
Our goal should be to keep the whole system running despite individual node failures, and to keep the impact of a node outage as small as possible.
#### Follower failure: Catch-up recovery
Follower keeps log of data changes received from leader, on it's local disk. In case of a failure or  restart, follower can recover by requesting all data changes after the last processed data change. It knows the last processed transaction from it's log.
#### Leader failure: Failover
Leader failure is handled using *failover*.
- One of the followers is promoted to be the new leader
- Clients need to be reconfigured to send writes to the new leader
- Other followers need to start consuming data changes from the new leader

Failover can be done *manually* or *automatically*. Automatic failover consists of following steps
1. *Determining that the leader has failed:* Since there is no fooproof way of knowing what has gone wrong with the node, most systems use timeout--nodes frequently bounce messages between each other and if node does not respond for some period of time, it is assumed to be dead.
2. *Choosing a new leader:* This could be done using *election process* or *controller node*. The best candidate for leadership is usually the one with most up-to-date data changes from the old leader.
3. *Reconfiguring the system to use the new leader:* Clients now need to send writes to the new leader. System needs to ensure that the old leader becomes a follower and recognizes the new leader.

Failover can go wrong fro many reasons
- *Lost writes:* If async replication is used, the new leader may not have received all writes from the old leader
- Discarding writes could be dangerous if other storage systems outside of the database need to be coordinated with database contents
- *Split brain:* Two nodes both believe they are the leader. If both leaders accept writes, and there is no process for resolving conflicts, data may be lost or corrupted.
- Suboptimal Timeout: A longer timeout means longer time to recovery.Too short a timeout can result in unnecessary failovers.                   

There are no easy solutions to these problems. So some teams prefer manual failover. The issues--node failures;unreliable networks;trade-offs around replica consistency, durability, availability ,and latency--are fundamental problems in distributed systems.
### Implementation of Replication Logs
#### Statement-based replication
Leader logs every write request(*statement*) that it executes and sends that statement log to followers. Each follower parses and executes the statements as if they had been received from client.
##### Problems with statement-based replication
- Any statement that calls a nondeterministic function is likely to generate a different value on each replica.
- Statements with autoincrementing columns or dependence on existing data in the database must be exeuted in exactly the same order. This is limiting for concurrent transactions.
- Statements with side effects(triggers, sprocs, user defined functions etc.) may have different effects on each replica, if the side effects are non-deterministic.

Other replications methods are preferred due to too many edge cases with statement replication.
#### Write Ahead Log(WAL) Shipping
In storage engines, typically every write is appended to a log e.g. SSTables, LSM Trees, B-Trees etc. Log is an append-only sequence of bytes containing all writes to the database.

We can use the exact same log to build a replica on another node: besides writing the log to a disk, the leader also sends it across the network to it's foillowers. When the follower processes this log, it builds the exact same copy of the data structures as found on the leader.

*Disadvantage of WAL Log:* The log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks. This makes replication closely coupled to the storage engine. If the database changes its storage format from one version to another, it is typically not posible to run different versions of the database on the leader and followers.
#### Logical(row-based) Log Replication
Uses different log formats for replication and for the storage engines, which allows the replication log to be decoupled from the storage engine internals.

A logical log for a relational database is usually a sequence of records describing writes to the database tables at the granularity of a row.
- For an inserted row, the log contains new values of all columns
- For a deleted row, the log contains enough information to uniquely identify the row that was deleted.
- For an updated row, the log contains enough information to identify the updated row, and new values for all the columns.

A transaction that modifies several rows generates several such log records, followed by a record indicting that the transaction was committed.

*Advantages of logical log:*
- Logical log can be more easily kept backward compatible as it is decoupled from storage engine internals.
- A logical log format is also easier for external applications to use. This is useful for sending database contents to an external system, such as data warehouse, indexes and caches. This technique is called *change data capture*.
#### Trigger-based replication
We may need to move replication up to the application layer from database system if:
- We want to replicate only a subset of data
- We want to replicate from one database system to another
- Or we need conflict-resolution logic

Some tools can make data changes available to the application by reading the database log or we can use *triggers* and *stored procedures*.

*Advantage of trigger-based replication:* Flexibility

*Disadvantages of trigger-based replication*
- Typically has greater overheads than other replication methods
- Is more prone to bugs and limitations than the database's built-in replication.
### Problems with Replication lag
*Read-scaling architecture*: Create many followers and distribute the read requests across those followers.

A fully synchronous replication to followers would be very unreliable. But at the same time asynchronous follower may fall behind, resulting in inconsistencies in data read by applications.

*Eventual consistency:* Data inconsistency due to the lag is just a temporary state. Followers will eventually catch up and become consistent with the leader. In practice, the replication lag may be only a fraction of a second, and not noticeable. However, in general there is no limit to how far the replica may fall behind.
#### Anomalies due Replication Lag 
##### Reading your own writes
Data is written to a leader, but can be read from a follower. With asynchronous replication, if the user views data shortly after it has been written, data may not have reached the replica. User may think they have lost the data.

In this situation we need **read-after-write consistency***(aka read-your-writes consistency)*.
- This is the guarantee that if user reloads the page, they will always see any updates they submitted themselves (i.e. their data is saved correctly).
- It makes no promises about the data written by other users.

*Implementing read-after-write consistency:*
- When reading soemthing user may have modified, read it from the leader. This requires knowing if something might have been modified, without actually querying it e.g. Always read a user's own profile from the leader.
- Track time of the last update, and for one minute after the update make all reads from the leader.
- Monitor the replication lag on followers and prevent queries on any follower that is more than one minute behind the leader.
- The client can remember the timestamp of it's  most recent write-- Then the system can ensure the read is handled by a replica that reflects updates until that timestamp or hold the query till the replica has caught up.
- For replicas distributed across multiple datacenters, request to be served by the leader must be routed to the datacenter hosting the leader.

**Cross-device read-after-write consistency:** User should see the information if they enter it on one device and view it on another device.
- Approaches like client remembering timestamp of last update break as one device is not aware of updates made by another device. This will need metadata to be centralized.
- For replicas distributed across different datacenters, there is no guarantee that connections from different devices will be routed to the same datacenter. If the approach requires reading from a leader, all the requests would have to be routed to the DC with the leader first.
##### Monotonic Reads
With asynchronous followers, users may see things *moving backward in time*, if the user makes several reads from different replicas.

*Monotonic reads* is a guarantee that if one user makes several reads in sequence, they will not see time go backward i.e. they will not read older data after having previously read new data. It's a lesser guarantee than strong consistency, but a stronger guarantee than eventual consistency.

One way of achieving it is by ensuring that each user always makes their reads from the same replica. Different users can read from different replicas.
##### Consistent Prefix Reads
*Violation of causality:* Records with causal dependency between them might not appear to be in the order of causality.

*Consistent prefix reads* is a guarantee that if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.

This is a particular problem in partitioned databases. In many distributed databases, different partitions operate independently, and there is no global ordering of writes.  When reading from a database, a user may see some parts of the database in older state and some parts in newer state.

One solution is to ensure that causally dependent writes are written to the same partiion.
#### Solutions for Replication Lag
*When working with an eventually consistent system, it is worth thinking about how the application behaves if the replication lag increases to several minutes or even hours.*
- If the answer is "no problem", that's great
- However, if the result is bad experience for the users, it's important to design the system to provide a stronger guarantee, such as read-after-write.

Pretending that replication is synchronous when in fact it is asynchronous is a recipe for problems down the line.

Dealing with these issues in application code is complex and easy to get wrong. It would be better if application developers didn't have to worry about the subtle replication issues and could just trust their database to "do the write thing". 

**Transactions** are the way for a database to provide stronger guarantees so that the application can be simpler.
- Single node transactions have existed for a long time
- In the move to distributed(replicated and partitioned) databases, many systems have abandoned transactions, claiming that transactions are too expensive in terms of performance and availability, and asserting that eventual consistency is inevitable in a scalable system. This may not be entirely true.
## Multi-leader replication(aka *master-master or active/active replication*)
In leader-based replication, writes must go through one leader making it a single point of failure.

*Multi-leader replication* allows more than one node to accept writes. In this setup, each leader simultaneously acts as a follower to other leaders.
### Use cases for multi-leader configuration
Benefits of using the multi-leader setup in a single datacenter rarely outweigh the added complexity.
#### Multi-datacenter operation
Each datacenter has a leader. 
- Within each datacenter, regular leader-follower replication is used.
- Between datacenters, each datacenter's leader replicates its changes to the leaders in other datacenters.

Comaparison of single and multi-Leader setups in a multi-datacenter deployment.
- **Performance**
    - In a single-leader configuration, every write must go over the internet to the datacenter with the leader. This can add significant latency to writes and contravene the purpose of having multiple datacenters.
    - In multi-leader configuration, every write can be processed in the local datacenter and asynchronously replicated to other datacenters. Thus the inter-datacenter network delays are hidden from the users, which means *perceived performance* may be better.
- **Tolerance of datacenter outages**
    - In single-leader replication, if the datacenter with the leader fails, a failover can promote a follower from another datacenter to be the leader.
    - In multi-leader replication, each datacenter can continue operating independently of others, and replication catches up when failed datacenter comes back online. 
- **Tolerance of network problems**
    - Traffic between the datacenters usually goes over the public internet, which may be less reliable than the local datacenter network.
    - A single leader configuration is very sensitive to problems in this inter-datacenter link, because writes are made synchronously over this link.
    - A multi-leader configuration with asynchronous replication can usually tolerate network problems better.
#### Clients with Offline Operation
We may have an application that needs to continue to work while it is disconnected from the internet. Any changes made when offline, need to be synced with the server and other devices, when we are back online. It typically works as follows:
- Every device has a local database that acts as a leader.
- There is asynchronous multi-leader replication process(sync) between the replicas on all the devices.
- The replication lag may be in hours or even days depending on when the internet access is available.

From an architectural point of view, this setup is essentially the same as multi-leader replication between datacenters taken to the extreme: *Each device is a datacenter, and the network connection between them is extremely unreliable*.
#### Collaborative Editing
*Real-time collaborative editing* applications allow several people to edit the document simultaneously.

When one user edits a document, changes are instantly applied to their local replica(the state of the document in their web browser or client application) and asynchronously replicated to the server and any other users who are editing the document.

To guarantee there will be no editing conflicts, the application must obtain a lock on the document before the user can edit it. Another user has to wait for editing the same document until the first user has committed their changes and released the lock. This collaboration model is equivalent to *single-leader replication with transactions on the leader*.

For faster collaboration, we may want to make the unit of change very small(e.g. a single keystroke) and avoid locking. This approach allows multiple users to edit simultaneously. This is equivalent to multi-leader replication.
### Disadvantages of Multi-Leader Replication
Multi-leader replication is often considered dangerous territory that should be avaoided if possible due to following reasons:
- The same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved.
- As multi-leader replication is a somewhat retrofitted feature in many databases, there are often subtle configuration pitfalls and surprising interactuons with other database features e.g. autoincrementing keys, triggers, and integrity constraints.
### Handling Write Conflicts
The biggest problem with multi-leader replication is that write conflicts can occur, which means that conflict resolution is required.
#### Synchronous vs asynchronous conflict detection
In single-leader database, the second writer will wait for the first writer to complete, or abort the second write transaction, forcing the user to retry the write.

In a multi-leader setup, both the writes are successful, and conflict is detected only at some later point in time.

*Synchronous conflict detection:* Wait for the write to be replicated to all the replicas before telling the user that the write was successful. However, this precludes allowing each replica to accept writes independently.
#### Conflict Avoidance
If the application can ensure that all the writes for a particular record go through the same leader, conflicts cannot occur. e.g. In an application where user can edit own data, we can ensure that requests from a particular user are routed to the same datacenter and leader. Different users may have different "home datacenters", but from a single user's perspective the configuration is essentially single-leader.

However, conflict resolution breaks down when we have to change the designated leader for a record, perhaps due to failure, rebooting etc.
#### Converging toward a consistent state
Every replication system must ensure that the data is eventually the same in all replicas. Thus the database must resolve the conflicts in a convergent way, which means that all replicas must arrive at the same final value when all changes have been replicated.

There are various ways of achieving convergent conflict resolution
- *Last Write Wins:* Give each write a unique ID, and pick the write with the highest ID as the winner. This is a popular approach, but is dangerously prone to data loss.
- Give each *replica* a unique ID, let writes originating at higher numbered replicas take precedance. This approach also implies data loss.
- Somehow, merge the values together e.g. Concatenation
- Record the conflict in an explicit data structure that preserves all information, and write application code that resolves the conflict at some later time(perhaps by prompting the user).
#### Custom Conflict Resolution Logic
As the most appropriate way of resolving the conflict may depend on the application, most multi-leader replication tools let you write conflict resolution logic using application code. The code may be executed on write or read:

*On write:* As soon as the database system detects a conflict in the log of replicated changes, it calls a conflict handler.

*On read:* When a conflict is detected, all the conflicting writes are stored. The next time the data is read, these multiple versions of the data are returned to the application. The application may prompt the user or automatically resolve the conflict., and write the result back to the database.

Conflict resolution usually applies at the level of an individual row or document, not for an entire transaction.
#### Automatic Conflict Resolution
Conflict resolution rules can quickly become complicated, and custom code can be error prone.

Research into automatically resolving conflicts
- *Conflict-free replicated datatypes(CRDTs)* are a family of data structures for sets, maps, ordered lists, counters etc. that can be concurrently edited by multiple users, and which automatically resolve conflicts in sensible ways.
- *Mergeable persistent data structures* track history explicitly(like the GIT version control system), and use a three-way merge function(whereas CRDTs use two-way merge)
- *Operational transformation* is the conflict resolution algorithm behind collaborative editing applications. It was designed particularly for concurrent editing of ordered list of items, such as the list of characters that constitute a text document.
#### What is a conflict?
- Two writes concurrently modified the same field in the same record.
- Two different records locking a common resource are entered on two different leaders
### Multi-leader Replication Topologies
A replication topology describes the communication paths along which writes are propagated from one node to another.
- *All-to-all topology:* The most general topology, in which every leader sends its writes to every other leader.
- *Circular topology:* Each node recieves writes from one node and forwards those writes(plus any writes of its own) to one other node.
- *Start topology:*One designated root node forwards writes to all of the other nodes. The star topology can be generalized to a tree.

To prevent infinite replication loops in circular and start topologies, each write is tagged with the identifiers of all nodes it has passed through. When a node receives a data change tagged with it's own identifier, that change is ignored.
#### Challenge with circular and star topologies: 
If one node fails, it can interrupt the flow of replication messages between other nodes. A more dense topology such as all-to-all has better fault tolerance, as the messages travel along different paths, avoiding a single point of failure.

#### Challenge with with all-to-all topologies
Some network links may be faster than others(e.g. due to network congestion), with the result that some replication messages may overtake others. This is a problem of causality. Simply attaching a timestamp to every write is not sufficient, because clocks cannot be trusted to be sufficiently in sync to correctly order these events at the leader.

*Version Vectors:* To order these events correctly, a technique called version vectors can be used.
## Leaderless replication
Single-leader and multi-leader replication are based on the idea that a client sends a write request to one node(the leader) and the database system takes care of copying that write to the other replicas.A leader determines the order in which the writes should be processed and followers apply the leader's writes in the same order.

*Leaderless replication* is an approach that allows any replica to accept writes from clients. Database using such style is also called *Dynamo-style*.

Leaderless implementation is typically designed in a couple of ways
- The client directly sends its writes to several replicas.
- A *coordinator node* sends writes to several replicas on behalf of the client. Unlike a leader database, the coordinator doesn't enforce any ordering of writes. This difference in design has a profound consequences for the way the database is used.
### Writing to the Database when a Node is Down
In a leaderless configuration, failover does not exist. If a node becomes unavailable, it might miss any writes that happened when it was down.  Once such node becomes available again, and starts accepting read requests, it may respond with stale data.

To solve that problem, client sends read request to several nodes in parallel. The client may get different values from different nodes i.e. up-to-date value from some nodes and stale values from some nodes. Version numbers are used to determine which value is newer.
#### Read repair and anti-entropy
To make data consistent across replicas, two mechanisms are often used in dynamo-style databases.

*Read Repair:* When a client makes read from several nodes in parallel, it can detect any stale values, and write back the newer values. This approach works well for values that are frequently read.

*Anti-entropy process*: A background process in datastores, that constantly looks for differences in the data between read replicas and copies any missing data from one replica to another.
- Anti-entropy process does not copy writes in any particular order 
- There may be a significant delay before data is copied
- Without an anti-entropy process, values that are rarely read may be missing from some replicas and thus have reduced durability.
#### Quorums for reading and writing
If there are *n* replicas, the write must be confirmed by *w* nodes to be considered successful, and we must query at least r nodes for each read. As long as *w+r > n*(at least one read and write nodes overlap), we expect to get an up-to-date value while reading because at least one of the *r* nodes we're reading from must be up-to-date. Reads and writes that obey these *r* and *w* values are called *quorum reads and writes*.

In Dynamo-style databases, the values for *n, w,* and *r* are typically configurable. A common choice is to make *n* an odd number(typically 3 or 5) and to set *w* = *r* = (*n*+1)/2.

There may be more than *n* nodes in the cluster, but any given value is stored only on *n* nodes.

The quorum condition, *w+r > n*, allows the system to handle unavailable nodes as follows.
- If *w < n*, we can still process writes if a node is unavailable
- If *r < n*, we can still process reads if a node is unavailable
- With n = 3, w = 2, r = 2 we can tolerate one unavailable node
- With n = 5, w = 3, r = 3 we can tolerate two unavailable nodes
- Normally writes are sent to all *n* replicas in parallel. The parameters *w* and *r* determine how many nodes we wait for- i.e. how many nodes need to report success before we consider the read or write to be successful.
- If fewer than *w* or *r* nodes are available, writes or reads return an error. Node could be unavailable for many reasons: crashed or powered down, eror executing the operation, network interruption between the client and the node. We only care if the node responded or not.
##### Limitations of Quorum Consistency
###### *w* + *r* <= *n*
Often *w* and *r* are chosen to be a majority(more than n/2) of nodes, but it may not always be the case. It only matters that the sets of nodes used for write and read overlap in atleast one node.

With a smaller *w* and *r* you are more likely to read stale values. On the upside, this configuration allows lower latency and higher availability.
###### *w* + *r* > *n*
- If a *sloppy quorum* is used,the *w* writes may end up on different nodes than *r* reads.
- If two writes happen concurrently,it is not clear which happened first. Only safe option is to merge the concurrent writes.
- If a write happens concurrently with a read, the write may be reflected on only some replicas. In this case, it's undetermined whether the read returns old value or new.
- If a write succeeded on some replicas and failed on others, and overall failed on less than *w* replicas, it is not rolled back on the replicas where it succeeded. This means if a write was reported as failed, subsequent reads may or may not return the value from that write.
- If a node carrying a new value fails, and it is restored from a node carrying the old value, the no. of replicas storing the new value may fall below *w*.
- There are edge cases where we may get unlucky due to timing.

Dynamo-style databases are generally optimized for "use cases that can tolerate eventual consistency". The parameters *w* and *r* allow us to adjust the probability of stale values being read, but are no absolute guarantees. *Stronger guarantees generally require transactions or consensus.*
##### Monitoring Staleness
From operational perspective, we should be aware of the health of our replication. If it falls behind significantly, it should alert us so that we can investigate the cause of the issue.

For leader-based replication, database typically exposes the metrics for replication lag, which we can feed into the monitoring system. This is possible because leaders and followers apply writes in the same order.

However, in systems with leaderless replication, there is no fixed order in which writes are applied, which makes monitoring more difficult. Instead, we could measure replica staleness and predict expected percentage of stale reads depending on the parameters n, w, r.

*Eventual consistency is a vague guarantee, but for operability its important to quantify "eventiual".*
##### Sloppy Quorums and Hinted Handoff
Databases with leaderless replication are appealing for *use cases that require high availability and low latency*, as they can tolerate:
- Failure of individual nodes without need for a failover
- Individual nodes going slow (because requests don't have to wait for all nodes to respond)
###### Sloppy Quorum
In a large cluster it's likely that the client can connect to some database nodes, but not to the nodes that it needs to assemble a quorum for a particular value

With a *sloppy quorum*: writes and reads still require successful *w* and *r* responses, but those may include nodes that are not among the desgnated *n* "home" nodes for the value.

Sloppy quorums are particularly useful for increasing write availability. However, it's only an assuarance of durability, namely that the data is stored on w nodes somewhere. There is no guarantee that a read of r nodes will see it until the hinted handoff has completed.  
###### Hinted Handoff
Once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate "home" nodes. This is called *hinted handoff*.
##### Multi-datacenter operation
Leaderless replication is also suitable for multi-datacenter operation, since it is designed to tolerate conflicting concurrent writes, network interruptions and latency spikes.
### Detecting Concurrent Writes
In Dynamo-style databases conflicts can occur due to concurrent writes to the same key and during read repair or hinted handoffs. 

The problem is that events may arrive in different order at different nodes, due to variable network delays and partial failures. If each node simply overwrote the value for the key whenever it received the write request from a client, the nodes would become permanently inconsistent.

In order to become eventually consistent, replicas should converge towards a common value. Most replicated databases do not handle this properly: to avoid losing data, application developers need to know a lot about the internals of database's conflict handling.
#### Last Write Wins(discarding concurrent writes)
When we say the writes are concurrent, their order is undefined.

Even though the writes don't have a natural ordering, we can forcean arbitrary order on them. e.g. attach a timestamp to each write, pick the biggest timestamp as more recent, and discard any writes with an earlier timestamp. This conflict resolution algorithm is called *last write wins*(LWW)

Example Implementations:
*Cassandra:* LWW is the only supported method for conflict resolution
*Riak:* LWW is an optional feature

LWW achieves the goal of eventual convergence, but at the cost of durability: if there are several concurrent writes to the same key, even if they were all reported as successful to the client(because they were written to *w* replicas), only one write will survive and rest will be silently discarded.

There are some situations, such as caching, in which lost updates are probably acceptable. If losing data is not acceptable, LWW is a poor choice for conflict resolution. The only safe way of using a database with LWW is to ensure that the key is written only once and thereafter treated as immutable, thus avoiding any concurrent updates to the same key.

An operation A *happens before* another operation B if B knows about A, or depends on A, or builds upon A in some way. Whether one operation happens before another operation is the key to defining what concurrency means. We can simply say that the two operations are concurrent if neither happens before the other(i.e. neither knows about the other). If one operation happened before another, the later operation shoudl overwrite the earlier operation, but if the operations are concurrent, we have a conflict that needs to be resolved.

**Concurrency Time and Relativity**
For two operations to be called concurrent, it is not important whether they literally overlap in time. We simply call two operations concurrent if they are bothe unaware of each other, irrespective of the physical time at which they occurred.

As per special theory of relativity, information cannot travel faster than light. Consequently, two events that occur some distance apart cannot possibly affect each other if the time between the events is shorter than the time it takes for the light to travel the distance between them.

In computer systems two events could be concurrent, even if the speed of light would in principle have allowed one operation to affect the other. For example, if the network was slow or interrupted at the time, two operations can occur some time apart and still be concurrent, because the network problems prevented one operation from lknowing about the other.
#### Capturing the happens-before relationship
Below is algorithm used to determine if two operations are concurrent or one happened before another
- The server maintains a version number for every key, increments the version number every time the key is written, and stores the new version number along with the value written.
- A client must read a key before writing. When client reads a key, the server returns all the values that have not been overwritten, as well as the latest version number.
- When a client writes a key, it must include the version number from the prior read, and it must merge together all values it received in the prior read.
- When the server receives a write with a particular version number, it can overwriteall values with that version number or below(since it knows that they have been merged into a new value), but it must keep all the values with a higher version number(because those values are concurrent with the incoming write).

When a write includes a version number from prior read, that tells us which previous state the write is based on.
#### Merging concurrently written values
Concurrently written values are also called *siblings* (in Riak). Merging siblings is essentially the same problem as conflict resolution in a multi-leader replication.

As merging siblings in application code is complex and error prone, there are some efforts to design data structures that can perform this merging automatically. e.g. Riak's datatype support uses a family of data structures called CRDTs that can automatically merge siblings in sensible ways,including preserving deletions.

Merging needs to use deletion markers called *tombstones* to mark removal of items while merging.
#### Version vectors
When there are multiple replicas accepting writes concurrently, we need to use a version number *per replica* as well as *per key*. Each replica increments its own version number when processing a write, and also keeps track of version numbers it has seen from each of the other replicas. This information indicates which values to overwrite and which values to keep as siblings.

The collection of version numbers from all the replicas is called a *version vector*. Riak2.o uses a variant of this idea called *dotted version vector*.

Version vectors are sent from replicas to clients when values are read, and need to be sent back to the database when value is subsequently written. The version vector allows database to distinguish between overwrites and concurrent writes.

The version vector structure ensures that it is safe to read from one replica and write back to another replica. Doing so may result in siblings being created, but no data is lost as long as siblings are merged correctly.






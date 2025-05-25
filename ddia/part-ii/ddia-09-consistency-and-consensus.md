# 9. Consistency and Consensus
**Best way of building fault-tolerant systems**
- Find some general-purpose abstractions with useful guarantees
- Implement them once, and 
- Then let applications rely on those guarantees. 

We need to **seek abstractions that** can **allow applications to ignore** some of the **problems with distributed systems**. 

One of the most important abstractions for distributed systems is **consensus**: that is **getting all the nodes to agree on something**. 
Reliably reaching consensus in spite of network faults and process failures is a surprisingly tricky problem.

We need to explore the range of guarantees and abstractions that can be provided in a distributed system. 
We need to understand the scope of what can and cannot be done: 
In some situations it is possible for a system to tolerate faults and continue working; in other situations that is not possible.

## Consistency Guarantees
Most **replicated databases** provide at least ***eventual consistency***:
- If we write to a database and wait for some unspecified length of time, then eventually all read requests will retrun the same value
- This is a very weak guarantee--it doesn't say anything about when the replicas will *converge*: e.g if we write a value and immediately read the value, there is no guarantee that we will see the value we just wrote, as the read may be routed to a different replica.
- **Eventual consistency** is **hard for application developers** because it is **so different from** the **behavior of variables in** a normal **single-threaded program**. A database has much more complicated semantics.

When working with databases that provide only weak guarantees, we need to be constantly aware of it's limitations and not accidentally assume too much
- Bugs are often subtle and hard to find by testing, because the application may work well most of the time.
- The edge cases of eventual consistency only become apparent when there is a fault in the system(e.g. a network interruption) or high concurrency.

Data systems may choose to provide **stronger consistency models**, but they **come at a cost**: systems with stronger guarantees may have worse performance or be less fault-tolerant than systems with weaker guarantees. 
**Stronger guarantees** can be appealing because they **are easier to use correctly**.

**Distributed consistency** is mostly about **coordinating the state of replicas in the face of delays and faults**.

## Linearizability
Linearizability is also known as atomic consistency, string consistency, immediate consistency, or external consistency. 

*The basic idea behind linearizability to make a system appear as if there were only one copy of the data, and all operations on it are atomic.* With this the application does not need to worry about multiple replicas in the system.

Linearizability is a *recency guarantee*: a linearizable system guarantees that the value read is the most recent, up-to-date value, and doesn't come from a stale cache or replica.
### What makes a system linearizable?
The requirement of linearizability is that, once a new value has been written or read, all subsequent reads(reads that started after th write was confirmed or reads that happen after the first read seein the new value) see the value that was written, until it is overwritten again.
- The order in which requests are processed matters rather than the order in which requests were made.
- If a write was processed, but the client making the write did not receive the response, it is ok for another client to read the new value.
- If a read from one client starts before the with write from another client is confirmed and finishes after the write is done, it is ok for such read to see the old value, as long as no other read has seen the new value.
- This model doesn't assume any transaction isolation: another client may change a value at any time.

It is possible(though computationally expensive) to test whether a system's behaviour is linearizable by recording the timings of all requests and responses, and checking whether they can be arranged in a valid sequential order.
### Linearizability vs Serializability
Linearizability and serializability are two quite different guarantees.

*Serializability*
- Serializability is an isolation property of transactions, where every transaction may read and write multiple objects.
- It guarantees that transactions behave the same as if they had executed in *some* serial order(each transaction running to completion before the next transaction starts).
- It is okay for that serial order to be different from the order in which transactions were run.

*Linearizability*
- Linearizability is a recency guarantee on the reads and writes of a register(an *individual object*).
- It doesn't group operations together into transactions, so it does not prevent problems such as write skew, unless we take additional measures such as materializing conflicts.

A database may provide both serializability and linearizability, and this combination is known as *strict serializability* or *strong one-copy serializability*.
- Implementations of serializability based on 2PL or actual serial execution are typically linearizable. 
- However, serializable snapshot isolation is not linearizable: by design, it makes reads from a consistent snapshot, to avoid lock contention between readers and writers. The whole point of a consistent snapshot is that it does not include writes that are more recent than the snapshot, and thus the reads from a snapshot are not linearizable.
### Relying on Linearizability
In few areas, linearizability is an important requirement for making a system work correctly.
#### Locking and leader election
In single-leader replucation, one way of electing a leader is to use a lock:every node that starts up tries to acquire the lock and the one that succeeds becomes the leader. Implementation of this lock should be linearizable: all nodes must agree which node owns the lock.

Coordination services like Apache ZooKeeper(with Apache Curator) and etcd are often used to implement distributed locks and leader elction. A linearizable storage service is the basic foundation for these coordination tasks.

Distributed locking is also used in some distributed databases, such as Oracle RAC.
#### Constraints and Uniqueness Guarantees
If we want to enforce hard uniqueness constraint while data is written to database, we require linearizability: a single up-to-date value that all nodes agree on. Business process cannot tolerate duplicates, as they are hard to fix later.

Other constraints such as foreign key or attribute constraints can be implemented without requiring linearizability. They can tolerate eventual consistency e.g. foreign key violations can be fixed through retries, notifications, or background jobs. Business process can tolerate eventual consistency for foreign keys.
#### Cross-channel timing dependencies
Cross-channel timing dependencies occur when different processes or systems in a distributed environment must coordinate actions based on the timing of events across channels.

If all systems use linearizable data stores, linearizability can:
- Ensure strong consistency for operations performed across services, databases, or message queues.
- Prevent issues like duplicate entries, stale reads, and out of order processing.
#### Implementing Linearizable Systems
*Single Leader Replication(potentially serializable)*:
- Leader has primary copy of data that is used for write, and followers maintain backup copies of data on other nodes.
- If we make reads from the leader, or from synchronously updated followers, they have potential to be linearizable.
- Not every single-leader database is linearizable, either by design(e.g. because it uses snapshot isolation) or due to concurrency bugs.

*Consensus algorithms(linearizable)*:
- Some consensus algorithms bear resemblance to single-leader replication
- Consensus algorithms can implement linearizable storage safely by using measures to prevent split brains and stale replicas.
- ZooKeeper and etcd work this way

*Multi-leader replication(not linearizable):*
- Systems with multi-leader replication are generally not linearizable, because they concurrently process writes on multiple nodes and asynchronously replicate them on other nodes.
- They can produce conflicting writes that require resolution
- Such conflicts are an artifact of the lack of a single copy of the data

*Leaderless replication(probably not linearizable):*
- LWW conflict resolution methods based on time-of-day clocks are almost certainly nonlinearizable, bcause clock timestamps cannot be guaranteed to be consistent with actual event ordering due to clock skew.
- Sloppy quorums also ruin any chance of linearizability.
- Even with strict quorums nonlinearizability is possible.

*Linearizability and quorums:*
- Dynamo-style quorums are not linearizable by default. If a write happens concurrently with read, the write may be reflected only on some replicas. In this case it is undetermined if a read returns old value or new.
- It is possible to make dynamo-style quorums linearizable at the cost of reduced performance: a reader must perform read repair synchronously before returning results to the application, and a writer must read the latest state of quorum of nodes before sending it' writes.
    - Riak does not perform synchronous read repair due to the performance penalty.
    - Cassandra does wait for read repair to complete on quorum reads, but it loses linearizability if there are multiple concurrent writes to the same key, due to it's use of LWW conflict resolution.
    - Only linearizable reads and writes can be implemented this way; a linearizable compar-and-set operation cannot because it requires a consensus algorithm.
- It is safest to assume that a leaderless system with Dynamo-style replication does not provide linearizability.
### Cost of Linearizability
#### CAP Theorem
The tradeoffs of linearizability are as follows
- If our application requires linearizability, and some replicas are disconnected from others due to a network problem, then some replicas cannot process requests while they are disconnected: they must either wait until network problem is fixed, or return an error(either way, they become unavailable).
-If our application does not require linearizability, then it can be written in a way that each replica can process requests independently, even if it is disconnected from other replicas. In this case, application can remain available in face of a network problem, but its behaviour is not linearizable.
 
Applications that don't require linearizability can be more tolerant of network problems. CAP is sometimes presented as *Consistency, Availability, Partition tolerance: pick 2 out of 3.*. A better way of phrasing CAP would be *either Consistent or Available when Partitioned*.

CAP as formally defined is of very narrow scope: it considers only one consistency model(namely linearizability), and one kind of fault(network partitions, or nodes that are alive but disconnected from each other). It doesn't say anything about network delays, dead nodes and other tradeoffs. Thus it has little practical value for designing systems. *CAP is best avoided:* There is a lot of misunderstanding and confusion around CAP, and it does not help us understand systems better.
#### Linearizability and network delays
Fault-tolerance is not the only reason to avoid linearizability. Liearizability i slow--and this is true all the time, not only during a network fault(e.g. multi-core CPUs reading and writing to memory).

Many distributed systems choose not to provide linearizability guarantees to increase performance.

If we want linearizability, the response time or read and write requests is at least proportional to the uncertainty of delays in the network.
- In a network with highly variable delays, like most computer networks, the response times of linearizable reads and writes is inevitably going to be high.
- Weaker consistency models can be mush faster, so this trade-off is important for latency sensitive applications.

## Ordering Guarantees
There is a deep connection between ordering, linearizability, and consensus
- Linearizability implies that operations are executed in some *well-defined order*.
- The main purpose of the leader in single-leader replication is to determine the *order of writes* in the replication log--that is the order in which the followers apply those writes.
- Serializability is about ensuring that transactions behave as if they were executed in some *sequential order*. 
- The timestamps and clocks are used to determine which one of the two writes happened later.
### Ordering and Causality
Ordering helps preserve causality. 
Examples of causality:
- Question must be asked before an answer
- A record must be created before it is updated
- Happened before relationship 
- In the context of Snapshot Isolation, *consistent snapshot* means *consistent with causality*:  observing the entire database at a point in time makes it consistent with causality: the effects of all operations that happened causally before that point in time are visible, but no operations that happened causally afterward can be seen. Read skew(non-repeatable reads) means reading data in a way that violates causality.
- SSI detects write skew by tracking causal dependencies between transactions.

Causality imposes ordering on events: cause comes before effect; a message is sent before that message is received; the question comes before the answer. 
- *Causal Order:* The chains of causally dependent operations define the *causal order* in the system i.e. what happened before what.
- *Causally Consistent System:* If a system obeys the ordering imposed by causality, we say that it is *causally consistent*.
#### The causal order is not a total order
A *total order* allows any two elements to be compared: so if we have two elements we can always say which one is greater and which one is smaller e.g. natural numbers are totally ordered

Mathematical sets are not totally ordered. We say they are *incomparable*, and therefore they are *partially ordered*.

The difference  between a total order and a partial order is reflected in different database concurrency models:
- *Lineraizability:* In a linearizable system we have *total order* of operations: for any two operations we can always say which happened first.
- *Causality:* The causality defines *partial order* of operations: some operations are ordered with respect to each other(if they are causally related), but some are incomparable(if they are concurrent) 

According to this definition, there are no concurrent operations in a linearizable datastore:there must be a single timeline along which all operations are totally ordered. There might be several requests waiting to be handled, but datastore ensures that every request is handled atomically at a single point in time, acting on a single copy of data, along a single timeline, without any concurrency.
#### Linearizability is stronger than causal consistency
*Linearizability implies causality*: any system that is linearizable will preserve causality correctly.
- The fact that linearizability implies causality is what makes linearizable systems simple to understand and appealing.
- However makinga system linearizable can harm it's performance and availability, especially if the system has significant network delays(e.g. it is geopraphically distributed).
- Some distributed databases have abandoned linearizability, whcih allows them to have better performance, but can make it difficult top work with them.

However, system can be causally consistent without incurring the performance hit of making it linearizable. Causal consistency is the strongest possible consistency model that does not slow down due to network delays and remains available in face of network failures.

In many cases, systems that appear to require linearizability in fact only really require causal consistency, which can be implemented more efficiently. Based on this observation researchers are exploring new kinds of databases that preserve causality, with performance and vaialability characteristics that are similar to those of eventually concistent systems.
#### Capturing causal dependencies
In order to maintain causality, we need to know whch operation *happened before* which operation.
- This is a partial order: concurrent operations may be processed in any order, but if one operation happened before another, then they must be processed in that order on every replica.
- Thus when, processing an operation, a replica must ensure that all causally preceding operations have have already been processed;if some preceding operation is missing, the later operation must wait until the preciding operation is processed.

In order to determine causal dependencies, we need some way of describing "knowledge" of a node in the system. If the node had already seen the value X when it issued the write Y, X and Y may be causally related.

Causal consistency needs to track causal dependencies across the entire database(all replicas). Version vectors can be generalized to do this.

In order to determine causal ordering, database needs to know which version of data was read by the application.
### Sequence Number Ordering
Keeping track of all causal dependencies can become impracticable, as explicitly tracking all the data that has been read would mean a large overhead.

Instead a better way is to use *sequence numbers* or *logical timestamps* to order events. *Logical clock* is an algorithm to generate sequence of numbers to identify operations, typically using counters that are incremented for every operation.
- Such sequence numbers are compact, and they provide total order. 
- In particular we can create sequence numbers in a total order that is *consistent with causality*. Such a total order captures all teh causality information, but also imposes more ordering than strictly required by causality.

In a database with single-leader replication, the replication log defines a total order of operations that is consistent with causality.
#### Noncausal sequence number generators
If there is not a single leader, it is less clear how to generate the sequence numbers for operations. Various methods are used in practice.
- Each node can generate its own independent set of sequemce numbers.
- Attach timestamps with sufficiently high resolution to each operation.
- Preallocated blocks of sequence numbers to each node.

These options are more performant and scalable compared to single-leader replication.They generate a unique, approximately increasing sequence number for each operation. 

However, the sequence numbers they generate are *not consistent with causality*, because they do not capture the ordering of operations across the nodes.
#### Lamport timestamps
Lamport timestamp is a simple method for generating sequence numbers that is consistent with causality.

Each node has a unique identifier, and each node keeps a counter of the number of operations it has processed. The Lamport timestamp is then simply a pair of(counter, node ID). Two nodes may sometimes have the same counter value, but by including the node id in the timestamp each timestamp is made unique.

Lamport timestamp provides *total ordering*. The key idea about Lamport timestamps, which makes them consistent with causality, is the following: 
- Every node and every client keeps track of the maximum counter value it has seen so far.
- When a node receives request or response with a maximum counter value greater than its own counter value, it immediately incfeases its own counter to the maximum.

As long as te maximum counter value is carried alongwith every operation, this scheme ensures that the ordering from the Lamport timestamps is consistent with causality, because every causal dependency results in an increased timestamp.

*Version Vectors vs Lamport timestamps:* They have a different purpose.
- *Version vectors* can distinguish whether two operations are concurrent or whether one is causally dependent on the other.
- Lamport timesstamps always enforce total ordering: from total ordering of lamport timestamps, we cannot tell whether two operations are concurrent or whether they are causally dependent.
#### Timestamp ordering is not sufficiently
Lamport timestamps by themselves are not enough to solve many common problems in distributed systems.

The total order of operations only emerges after we have collected all of the operations. 
- If another node has generated some operations, but w don't yet know what they are, we cannot construct a final ordering of operations.: the unknown operations from other nodes may have to be inserted in various positions in the total order.
- In order to implement something like a uniqueness constraint of usernames, it's not sufficient to have total order of operations--we also need to know when that order is finalized.

The idea of knowing when total order is finalized in captured in the topic of *total order broadcast*.
### Total Order broadcast(Atomic broadcast)
A single-leader replication determines total order of operations by choosing one node as the leader and sequencing all operations on the single CPU core of the leader. The challenge is then how to scale the system if throughput is greater than a single leader can handle, and also how to handle failover in case leader fails. In the distributed systems literature this problem is known as *total order broadcast or atomic broadcast.*

*Scope of ordering Guarantee:* Partitioned databases with single leader per partition often maintain ordering only per partition, which means they cannot offer consistency guarantees(e.g. consistent snapshots, foreign key constraints etc.) across partitions. Total ordering across all partitions is possible but requires additional coordination.

Total order broadcast is usually described as a protocol for exchanging messages between nodes. Informally, it requires that two safety properties always be satisfied:
- *Reliable delivery:* If message is delivered to one node, it is delivered to all nodes.
- *Totally ordered delivery:* Messages are delivered to every node in teh same order.
#### Using total order broadcast
Consensus services such as ZooKeeper and etcd actually implement total order broadcast.

Total order broadcast can be used to implement:
- Database replication:
    - If every message represents a write to the database, and every replica processes the same writes in the same order, then the replicas will remain consistent with each other.
    - This principle  is known as *state machine replication*.
- Serialization
    - If every message represents a deterministic transaction to be executed as a stored procedure, and if every node processes those messages in the same order, then the partitions and replicas of the database are kept consistent with each other.
- Lock service that provides fencing tokens
    - Every request to acquire the lock is appended as a message to the log, and all messages are sequentially ordered in the order they appear in the log.
    - The sequence number can then serve as a fencing token, because it is monotonically increasing.
    - In ZooKeeper, this sequence number is called the zxid.


An important aspect of total order broadcast is that the order is fixed at the time the messages are dellivered: a node is not allowed to retroactively insert a mesage in an earlier position in the order if subsequent messages have already been delivered.

Another way of looking at total order broadcast is that it is a way of creating a log(as in replication log, transaction log, write-ahead log):
- Delivering a message is like appending to the log
- Since all nodes must deliver the same messages in the same order, all nodes can read the log and see same sequence of messages.
#### Implementing linearizable storage using total order broadcast
There are close links between linearizability and total order broadcast, but they are not the same.
- *Total order broadcast is asynchronous:* messages are guaranteed to delivered reliably in a fixed order, but there is no guarantee about *when* a message will be delivered(so one recipient may lag behind the others).
- *By contrast, linearizability is a recency guarantee:* a read is guaranteed to see the latest value written.

However we can buold a linearizable storage on top of total order broadcast e.g we can implement linearizable compar-and-set operation by using a total order broadcast as an append-only log.
#### Implementing total order broadcast using linearizable storage
- Assume we have a linearizable that stores and integer and has an atomic increment-and-get operation.
- For every message we want to send through total order broadcast, we increment-and-get the linearizable integer, and then attach the  value we got from the register as sequence number to the message. 
- We can then send the message to all nodes, and recipients will deliver the message consecutively by sequence number.

The key difference between tptal order broadcast and timestamp ordering is: the numbers we get from linearizable register form a sequence with no gap.

It can be proved that linearizable compare-and-set(or increment-and-get) register and total order broadcast are both equivalent to *consensus*. If we can solve one of these problems, we can transaform it into a solutio for others.

## Distributed Transactions and Consensus
Consensus is one of the most important and fundamental problems in distributed computing.

Informally the goal of consensus is to: *get several nodes to agree on something*. There are number of situations in which it is important for nodes to agree:
- *Leader election:* In a database with single-leader replication, all nodes need to agree on who is the leader to avoid split=-brain situation.
- *Atomic commit:* 
    - In a database that supports transactions spanning several nodes or partitions, we have the problem that 
    transaction may fail on some nodes but succeed on others.
    - If we want to maintain transaction atomicity, we have to get all nodes to agree on the outcome of a transaction. either they all abort/rollback or they all commit
### Atomic  Commit and Two-Phase Commit(2PC)
Atomicity prevents failed transactions from littering the database with half-finished results amd half-updated state. This is especially important in multi-object transactions and databases that use secondary indexes.
#### From single-node to distributed atomic commit
On asingle node, transaction commitment depends on the *order* in which data is durably written to a disk: first the data, then commit record. The key deciding moment for whether the transaction commits or abort is the moment at which disk finishes writing the commit record. Thus, it is a single device(the controller of one particular disk drive, attached to one of the nodes) that makes the commit atomic.

If multiple nodes are involved in a transaction (e.g. multi-object transaction in a partitioned database, or term partitioned secondary index(in which case index entry might be on a different node than primary data)), it is not sufficient to send commit request to all of the nodes and independently commit the transaction on each node. 
- It could very well happen that commit succeeds on some nodes and fails on other nodes.
- If some nodes commit the transaction while others abort it, the nodes become inconsistent with each other.
- Once a transaction has been committed on a node, it cannot be retracted.
- Node must only commit when it is certain that all other nodes in the transaction are also going to commit.

A transaction commit must be irrevocable.
- The reason for this rule is that once data has been committed, it becomes visible to other transactions, and thus other clients may start relying on that data.
- If a transaction was allowed to abort after committing, any transactions that read the data would be based on the data that was retroactively decalred to have not existed--so they have to be reversed as well.
#### Introduction to two-phase commit
Two-phase commit is an algorithm for achieving atomic transaction commit across multiple nodes-i.e. to ensure that all nodes commit or all nodes abort.

2PC is used internally by some distributed databases and also made available to applicatuons in the form of XA transactions(which are supported by Java Transaction API) or via WS-AtomicTransaction for SOAP web services.

2PC uses a component called *coordinator*(aka *transaction manager*). 
- The coordinator is often implemented as a library within the same application process that is requesting the transaction(e.g. embedded in a Java EE container).
- But it can also be a sepearate process or service
- Examples of coordinators: Narayana, JOTM, BTM, oe MSDTC

How does 2PC work?
- A distributed transaction begins with the application reading and writing data on multiple database nodes(called *participants*).
- Phase 1:
    - When the application is ready to commit, the coordinator sends *prepare requests* to each participant, asking them whether they are able to commit.
    - Nodes respond to prepare request with "yes" or "no".
- Phase 2:
    - If all participants reply with "yes", the coordinator sends commit request to all participants, and the actual commit takes place.
    - If any of the participants replies "no", the coordinator sends an abort request to all nodes.
##### A system of promises
2PC process in bit more detail
1. The application requests a globally unique transaction id from coordinator.
2. The application begins a single-node transaction on each participant, and attaches the globally unique transactionId to the single-node transaction. If anything goes wrong at this stage(e.g. a node crashes or request times out), the coordinator or any of the participants can abort.
3. When the application is ready to commit, the coordinator sends a prepare request to all participants, tagged with the global transaction ID. If any of these requests fails or times out, the coordinator sends an abort request for that transaction ID to all participants.
4. When a participant receives prepare request, it makes sure that it can definitely commit transaction under all circumstances.
    - This includes writing all transaction data to disk, and checking for any conflicts or constraint violations.
    - By replying "yes" to the coordinator, the node promises to commit the transaction without error if requested.
    - In other words, the participant surrenders the right to abort the transaction, biut without actually committing it.
5. On receiving all the responses, teh coordinator makes a definitive decision on whether to commit or abort the transaction. The coordinator must write that decision to its transaction log on the disk. This is called the *commit point*.
6. Once the coordinator's decision is written to disk, the commit or abort request is sent to all participants.
    - If this request fails or times out, coordinator must retry forever until it succeeds.

The protocol contains two cricial points of no return
1. When participant has voted "yes", it promises that it will definitely be able to commit later(although coordinator may still choose to abort).
2. Once the coordinator decides, that decision is irrevocable.
#### Coordinator failure
The only way 2PC can complete is by waiting for the coordinater to recover.
- If the coordinator fails before sending the prepare request, a participant can safely abort the transaction.
- But once the participant has received the prepared request and voted "yes", it can no longer abort unilaterally--it must wait to hear back from the coordinator whether it should commit or abort.
- If the coordinater crashes or network fails at this point, the participant can do nothing but wait. A participan't transaction in thois case is called *in doubt or uncertain*.
- Coordinator must write it's decision to it's transaction log before sending commit or abort requests to participants.

The commit point of 2PC comes down to a regular single-node atomic commit on the coordinator.
#### Three Phase Commit
Two-phase commit is called a blocking atomic commit protocol due to the fact that 2PC can become stuck for the coordinater to recover.

As an alternative to 2PC, an algorithm called *three-phase commit*(3PC) has been proposed as a nonblocking atomic commit protocol. However 3PC assumes a network with bounded delay and nodes with bounded response times. In general, nonblocking atomic commit requires a *perfect failure detector* i.e. a reliable mechanism for tellin whether a node has crashed or not. In anetwork with unbounded delay, timeout is not a reliable failover detector, because a request may timeout due to a network problem even if node has not crashed. So 2PC continues to be used in practice.
### Distributed Transactions in practice
Distributed transactions, especially those implemented with 2PC are criticized for causing operational problems, killing performance and promising more than they can deliver.
- Many cloud services choose not to implement distributed transactions due to the operational problems they engender
- Some implementations of distriobuted transactions carry a heavy performance penalty
- Most of the performance cost inherent in 2PC is due to disk forcing(fsync) that is required for crash recovery, and the additional network round trips.

Types of distributed transactions
- *Database-internal distributed transactions:* 
    - Some distributed databases(i.e. databases that use replication and partitioning in their standard configuration) support internal transactions among the nodes of that database
    - In this case all the nodes participating in the transaction are running the same database software
    - e.g. VoltDB and MySQL Clusters NDB storage engine
    - Database-internal transactions do not have to be compatible with any other system, so they can use any protocol and optimizations specific to that technology. They can often work quite well.
- *Heterogenous Distributed Transactions:*
    - Ina heterogenous transaction, participants are two or more different technologies: for example two databases from different vendors, or even non-database systems like message brokers.
    - Transactions spanning heterogenous technologies can be quite challenging.
#### Exactly-one message processing
A message from a message queue can be acknowledged as processed if and only if the database transaction for processing the message was successfully committed. 
- This is implemented by atomically committing the message acknowledgement and the database writes in a single transaction.
- If the message delivery, or the database transaction fails, both are aborted, and so the message broker may safely redeliver the message later.
- Thus by atomically committing the message ad the side effects of its processing, we can ensure that message is effectively processed only once, even if it required few retries before it could succeed.
- Such a distributed transaction is only possible if all the systems affected by the transaction are able to use the same atomic commit protocol.
#### XA-transactions
*X/Open XA(short for eXtended Architecture)* is a standard for implementing 2PC across heterogenous technologies.

XA is supported by:
- Many traditional databases: including PostgreSQL, MySQL, DB2, SQL Server and Oracle
- Message brokers: ActiveMQ, HornetMQ, MSMQ, IBM MQ

XA is notb a network protocol--it is merely a C API for interfacing with a transaction coordinator
- Bindings for this API exist in other languages.
- In JavaEE applications, XA transactions are implemented using JTA, which in turn is supported by: 
    - Many drivers for databases using JDBC
    - Many drivers for message brokers using JMS APIs

XA assumes that your application uses a network driver or client library to communicate with the participants.
- The driver calls the XA API to find out if an operation should be part of distributed transaction--and if so, it sends the necessary information to the database server.
- The driver also exposes the callbacks through which coordinator can ask the participant to prepare, commit or abort.

In practice, the coordinator is often simply is a library that is loaded into the same process as the application issuing the transaction. It keeps track of participants in the transaction,collects participants reponses after asking them to prepare, and uses log on the local disk to keep track of the commit/abort decision for each transaction.
#### Holding locks while in doubt
Database transactions usually take row-level exclusive locks on any rows they modify, to prevent dirty writes. In addition, if we want to use serializable isolation, a database using 2PL would also have to take sared lock on any rows read by the transaction.

The database cannot release those locks until the transaction commits or aborts. Thus, when using 2PC, the transaction must hold onto the locks throughout the time it is in doubt.

Whike those locks are held, no other transactions can modify those rows or even reading those rows. This can cause large parts of our database to become unavailable, util the in doubt transaction is resolved.
#### Recovering from coordinator failure
- In practice, *orphaned* in doubt transactions do occur(e.g. the coordinator transaction log has been lost or corrupted due to a software bug).
- These transactions cannot be resolved automatically, so they sit forever in the database, holding locks and blocking other transactions.
- Even rebooting database servers will not fix this, since since a correct implementation of 2PC must preserve the locks of in doubt transactions even across restarts.
- The only way is for an administrator to manually decide to whether commit or rollback the transactions.
- Many XA implementations have an escap hatch called *heuristic decisions:* allowing a participant to unilaterally decide to commit or abort an in  doubt transaction without a definitive direction from the coordinator. *Heuristic* here is an euphemism for *probably breaks consistency*.
#### Limitations of distributed transactions
Though useful, XA transactions can inroduce major operationsl problems. The key realization is that the transaction coordinator is itself a kind of database(in which transactions outcomes are stored).
- Coordinator is not replicated in many databases, thus becoming a single point of failure for the entire system(since its failure causes other application servers to block on the locks held by -n-doubt transactions)
- When the coordinatopr is part of the application server, suddenly the coordinator's logs become a crucial part of the durable system state. Such application servers are no longer stateless.
- XA is necessarily a lowest common denominator, as it needs to be compatible with a wide range of systems. It does not detect deadlocks across different systems and it does not work with SSI.
- For 2PC to successfully commit a transaction, all nodes must respond. If any part of the system is broken, any transaction also fails.Distributed transactions have a tendency of *amplifying failures*, which runs counter to our goal of building fault-tolerant systems.
### Fault Tolerant Consensus
The consensus problem is formalized as follows: one or more nodes may *propose* values,and the consensus algorithm *decides* on one of those values.

In thios formalism, consensus algorithm must satisfy the following properties:
- *Uniform agreement:* No two nodes decide differently
- *Integrity:* No node decides twice
- *Validity:* If a node decides value *v*, then *v* was proposed by some node
- *Termination:* Every node that does not crash eventually decides a value

The uniform agreement and integrity properties define the core idea of consensus: everyone decides on the same outcome, and once you have decided, you cannot change your mind. 

The validity property exists to mostly rule out trivial solutions. 

The termination property formalizes the idea of fault tolerance. It essentially says that consensus algorithm must make progress. Even if some nodes fail, the other nodes must reacha decision. Termination is a liveness property, while other three are safety properties.

The system model of consensus assumes that when a node "crashes", it suddenly disappears and never comes back. in this system model, any algorithm that has to wait for a node to recover is not going to satisfy the termination property. - - - There is a limit to the number of node failures that an algorithm can tolerate. Consensus algorithm requires at least majority of nodes to be functioning correctly in order to asure termination.
- Most implementations of consensus ensure that safety properties are always met, even if majority nodes fail or there is a severe netwok problem. Thus, a large scale outage can stop the system from being able to process requests,but it cannot corrupt the consensus system by causing it to make invalid decisions.

Most consensus algorithms assume that there are no Byzantine faults. That is, if the node does not correctly follow the protocol(e.g. by sending contradictory messages to different nodes), it may break the safety properties of the protocol.
#### Consensus algorithms and total order broadcast
The best-known fault-tolerant consensus algorihms are
- Viewstamped Replication
- Paxox
- Raft
- Zab

Most of these algorithms decide on *sequence* of values, which makes them *total order broadcast* algorithms.
- Total order broadcast requires messages to be delivered excatly once, in the same order to all the nodes. This is equivalent to performing several rounds of consensus: in each round the nodes propose the message they want to send next, and then decide on the next message to be delivered in the total order.
- Total order broadcast is equivalent to several rounds of consensus(each consensus decision corresponds to one message delivery)
    - Due to the agreement property of consensus, all nodes decide to deliver the same messages in the same order.
    - Due to the integrity property, messages are not duplicated
    - Due to the validity property messages are not corrupted and not fabricated out of thin air.
    - Due to the termination property, mesages are not lost.

Viewstamped replication, Raft and Zab implement total order broadcast directly, because that is more efficient than doing repeated rounds of one-value-at-a-time consensus. In teh case of Paxos, this optimization is known as Multi-Paxos.
#### Single leader replication and consensus
Single-leader replication takes all the writes to the leader and applies them to the followers in the same order.

In some databases, the leader is manually selected and configured by the humans in the operations team. Such a system does not satisfy the termination property of consensus because it requires human intervention in order to make progress.

Some databases perform automatic leader election and failover, promoting a follower to be the new leader if the old leader fails. However tyo avoid split brain, we need consensus to select a leader.
#### Epoch Numbering and quorums
All the consensus protocols discused so far internally use a leader in one form or another, but they don't make a guarantee that the leader is unique. Instead they can make a weaker guarantee: the protocols define an *epoch number* and guarantee that within each epoch, the leader is unique.

Every time a current leader is thought to be dead, a new vote is started among nodes to elect a new leader. This election is given an incremented epoch number and thus epoch numbers are totally ordered and monotonically increasing. If there is a conflict between two leaders from two different epochs, then the leader with higher epoch wins.

For every decision that a leader wants to make, it must send the proposed value to the other nodes and wait for a quorum of nodes to respond in favor of the proposal. A node votes in favor of the proposal only if it is not aware of any other leader with a higher epoch.
 
 Thus we have two rounds of voting:
 1. Once to choose a leader
 2. A second time to vote on leader's proposal

The key insight is that the quorums for those two votes must overlap: if a vote on a proposal succeeds, at least one of th nodes that voted for itmust have also participated in the most recent leader election. If teh vote for proposal doesn't reveal any higher numbered epoch, the current leader can then safely decide the proposed value.

Following differences to 2PC are key to correctness and fault-tolerance of a consensus algorithm
- In 2PC, coordinator is not elected
- Fault-tolerant consensus algorithms only require votes fom majority of nodes, whereas 2PC requires yes vote from every participant.
- Consensus algorithms define a recovery process by which nodes can get into consistent state after a new leadr is elected, ensuring that the safety properties are always met.
#### Limitations of Consensus
Consensus algorithms are a huge breakthrough in distributed systems:
 -They bring concrete safety properties(agreement, integrity, and validity) to systems where everything else is uncertain.
 - They remain fault-tolerant(able to make progress as long as majority of nodes are working and are reachable)
 - They provide total order broadcast, and therefore can implement linearizable atomic operations in a fault-tolerant way.

 Benefits of consensus algorithms come at a cost
 - The process by which nodes vote on proposals before they are decided is kind of synchronous replication.
 - Consensus systems always require a strict majority to operate.
    - This means we need minimum of 3 nodes to tolerate 1 failure and minimum 5 nodes to tolerate 2 failures.
    - If a network failure cuts off some of the nodes from rest, only the majority portion of the network can make progress, and the rest is blocked
- Most consensus algorithms follow static membership: they assume a fix set of nodes that participate in voting, which means that we cannot just add or remove nodes in a cluster. *Dynamic memberships extensions* are less understood and uncommon.
- Consensus systems generally rely on timeouts to detect failed nodes. In environments with highly variable network delays, especially geographically distributed systems, it often happens that a node falsly believes a leader has failed due to a transient network issue.Frequent leader elction can rsult in a terrible performance.
- Sometimes, consensus algorithms are particularly sensitive to network problems.

### Membership and Coordination services
Projects like *ZooKeeper* and *etcd* are often described as "distributed key-value stores" or "coordination and configuration services". Systems like HBase, Hadoop Yarn, Openstack Nova, and Kafka all rely on ZooKeeper running in the background.

ZooKeepr and etcd are designed to holds mall amounts of data that can fit entirely in memory(although they still write to disk for durability). That small amount of data is replicated across all the nodes using a fault-tolerant total order broadcast algorithm.

ZooKeeper is modeled after Google's chubby locking service. In addition to total order broadcast( and hence consensus), ZooKeeper has interesting features that are useful when building distributed systems.
- *Linearizable atomic operations*
    - Using an atomic compar-and-set operation we can implement a lock: if several nodes concurrently try to perform the same operation, only one of them will succeed
    - The consensus protocol guarantees that the operation will be atomic and linearizable, even if a node fails or network is interrupted at any point.
    - A distributed lock is usually implemented as a *lease*, which has an expiry time so that it is eventually released if a client fails.
- *Total ordering of operations*: Zookeeper provides a *fencing token*, by totally ordering all operations and giving each operation a monotonically increasing transaction ID(zxid) and version number(cversion)
- *Failure Detection*
    - Clients maintain a long-lived session on ZooKeeper servers.
    - The client and server periodically exchange hearbeats to check that the other node is still alive.
    - Even if the connection is temprarily interrupted or ZooKeeper node fails, the session remains active.
    - However, if the heartbeats cease for a duration that is longer than the session timeout, ZooKeeper declares the session to be dead.
    - Any locks held by a session can be configured to be automatically released when the session times out.
    - ZooKeeper calls these *ephemeral nodes*.
- *Change Notifications*
    - One client can read the locks and values created by other clients, and also watch them for changes.
    - By subscribing to notifications, a client avoids having to frequently poll to find out about changes.

Out of these features, only linearizable atomic operations require consensus.However, it is the combination of these features that makes systems like ZooKeeper, so useful for distributed coordination.
#### Allocating work to nodes
Examples where ZooKeeper/Chubby model works well
- If we have several instances of a process or service, and one of them needs to be chosen as a leader or primary.
    - If one node fails another should tyake overhead
    - e.g. single-leader databases, job schedulers and similar stateful systems.
- In a partitioned system, to decide which partition to assign to which node
    - Rebalancing partitions when new nodes join the cluster, or nodes are removed or fail.
    - e.g. partitioned database, message broker, file storage, distributed actor system etc.

These kind of tasks can be achieved by judicious use of atomic operations, ephemeral nodes, and notifications in ZooKeeper. If done correctly, this approach allows the application to automatically recover from faults without human intervention.

For an application running on thousands of nodes, performing majority votes on so many nodes would be terribly inefficient. Instead ZooKeeper runs on a fixed number of nodes(usually 3 or 5) and performs it's majority votes among these nodes while supporting a large number of clientsThus, Zookeeper provides a way of outsourcing some of the work of coordinating nodes(consensus, operation ordering, and failure detection) to an external service.

Normally the kind of data managed by ZooKeeper is quite slow changing: it represents information like "the node running on IP Address 10.1.1.23 is the leader for partition 7", and such assignments usually change on a timescale of minutes or hours.

For replicating application state, which may chnge thousands or millions of times per second, other tools(such as Apache BookKeeper) can be used.
#### Service Discovery
ZooKeeper, etcd, and Consul are also often used for *service discovery*--that is to find out which IP address we need to connect to in order to reacha aparticular service.
- In cloud datacenter envoronments, where it is common for virtual machines to come and go, we often don't know the IP addresses of our services ahead of time.
- Instead we can configure our services such that when they start up they register their network endpoints in a service registry, where they can be found by other services.

However, it is less clear of service discovery requires consensus. Reads from a DNS are absolutely not linearizable, and it is usually not considered problematic if the results from a DNS query are a little stale. It is more important that DNS is reliably available and robust to network interruptions.

If the consensus system already knows who the leader is, then it can make sense to also use that information to help other services discover who the leader is. Some consensus systems support read only caching replicas. These replicas asynchronously receive the log of all decisions of the consensus algorithm, but do not actively participate in voting. They are therefore able to serve read requests that do not need to be linearizable.
#### Membership services
A membership service determines which nodes are currently active and live members of a cluster. If we couple failure detection with consensus, nodes can come to an agreement about which nodes should be considered alive or not.

It could still happen that a node is incorrectly declared dead by consensus, nodes can come to an agreement about which nodes should be considered alive or not.






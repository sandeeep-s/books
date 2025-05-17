# Transactions
Some authors have claimed that general two phase commit is too expensive to support, because of the performance or availability problems that it brings. We believe it is better to let application programmers deal with the performance problems due to overuse of transactions as the bottlenecks arise, rather than always coding around the lack of transactions.

In the harsh reality of data systems, many things can go wrong.
- The database software or hardware may fail at any time(even in middle of a write operation)
- The application may crash at any time(including halfway through a series of operations)
- Interruptions in the network can unexpectedly cutoff application from the database, or one database npode from another.
- Several clients may write to the database at the same time, overwriting each other's changes
- A client can read data that doesn't make any sense because the data has only partially been updated
- Race conditions between clients can cause surprising bugs

In order to be reliable, a system has to deal with these faults and ensre that they don't cause a catstrophic failure of the entire system.

*Transactions* are a way for simplifying these issues.
- A transaction is a way for an application to group several reads and writes together into a logical unit.
- Conceptually, all the reads and writes in a transaction are executed as one operation: either the transaction succeeds(*commit*) or it fails(*abort, rollback*).
- If a transaction fails, application can safely retry.
- Withe transactions, error handling becomes much simpler for an application, because it doesn't need to worry about partial failure.

Transactions were created to *simplify the programming model* for applications accessing the database: 
- Transactions provide certain *safety guarantees*.By using tarnsactions, application is free to ignore certain potential error scenarios and concurrency issues, because the database takes care of them instead
- Not every application needs all the safety guarantees provided by transactions, and sometimes there are advantages to weakening transactional guarantees or abandoning them entirely(for example to achieve better performance or availability)
- Some safety guarantees can be achieved without transactions.

*How do we figure out if we need transactions?:* To answer that question, we first need to understand what safety guarantees transactions can provide, and what costs are associated with them.

## The Slippery Concept of a Transaction
Almost all relational databases today, and some non-relational databases support transactions.

There is a popular belief that transactions are the antithesis of scalability, and that any large-scale system would have to abandon transactions in order to maintain good performance and high availability.
On the other hand transactional guarantees are sometimes presented by database vendors as an essential requirement for "serious applications" with "valuable data".

Both viewpoints are pure hyperbole.
- Like every other technical design choice, transactions have advantages and limitations(trade-offs).
- In order to understand those trade-offs, we need to go into the details of the guarantees that transactions can provide--both in normal operation and various extreme(but realistic) circumstances.
### The Meaning of ACID
The safety guarantees provided by transactions are often described by the acronym ACID. 

In, practice, one database's implementation of ACID does not equal another's implementation. today, when a system claims to be "ACID compliant", its unclear what guarantees we can expect.

Systems that do not meet ACID criteria are sometimes called BASE (i.e Basic Availability, Soft state, and Eventually Consistent).
#### Atomicity
ACID atomicity describes what happens if a client wants to make several writes, but a fault occurs after some of the writes have been processed. If the writes are grouped together in an atomic transaction, and the transaction cannot be completed(*committed*) due to a fault, then the transaction is *aborted* and the database must discard or undo any writes it has made so far in that transaction.

Without atomicity, if an error occurs partway through making multiple changes, it's difficult to know which changes have taken effect and which haven't. The application could try again, but that risks making the same change twice, leading to duplicate or incorrect data. Atomicity simplifies this problem: if a transaction was aborted, the application can be sure that it didn't change anything, so it can be safely retried.
#### Consistency
The word Consistency is overloaded
- *Replica consistency* and *eventual consistency* in replication
- *Consistent hashing* is an approach to partitioning that some systems use for rebalancing
- In the CAP theorem, the word *consistency* is used to meam *linearizability*.

In te context of ACID, consistency refers to an application specific notion of being in a good state. 
- The idea of ACID consistency is that you have certain statements about your data(invariants) that must always be true.
- If a transaction starts with a database that is valid according to these invariants, and any writes during the transaction preserve this validity, then you can be sure that the invariants are always satisfied.
- This idea of consistency depends on the applications notion of invariants, and it's the application's responsibility to define the transactions correctly so that they preserve consistency.
- This is not something database can guarantee: if we write bad data which violates our invariants, the database can't stop us.

Atomicity, isolation, and durability are properties of the database, whereas consistency(in the ACID) sense is a property of the application.
#### Isolation
Isolation in the sense of ACID means that concurrently executing transactions are isolated from each other: they cannot step on each other's toes.

Isolation is formalized as *serializability*, which means that each transaction can pretend that it is the only transaction running on the entire database. The database ensures that, when the transactions have committed,the result is the same as if they had run *serially* , evn though in reality they may have run concurrently.

Serializable isolation is rarely used in practice, as it carries a performance penalty.
#### Durability
Durability is the promise that once the trasaction has committed successfully, any data that it has written is not forgotten, even if there is a hardware fault or database crashes. 
- In a single-node database, durability typically means that the data has been written to a non-volatile storage such as a hard drive or SSD.It usually also involves a write ahead log or similar, which allows recovery in teh event that teh data structures on the disk are corrupted.
- In a replicated database, durability may mean that the data has been copied to some number of nodes.

In order to provide a durability guarantee, a database must wait until these writes or replication s are complete, before reporting a transaction as successfully committed.

**Replication and Durability:** Perfect durability does not exist wheter with disks, SSDs or replication.
- If you write to disk and the machine dies, even though your data isn't lost, it is inaccessible until you fix the machine or transfer the disk to another machine. Replicated systems can remain available
- A correlated fault--a power outage or a bug that carshes every node on a particular input--can knock out all replicas at once, losing any data that is only in memory. Writing to disk is therefore still relevamt for in-memory databases.
- In an asynchronously replicated system, recent writes may be lost if the leader becomes unavailable.
- Refer book for more....

In practice there is no one technique that can provide absolute guarantees. There are only various risk-reduction techniques, including writing to disk, replicating to remote machines, and backups--and they can and should be used together.

### Single and Multi-object Transactions
Multi-object transactions modify several objects (records, rows or documents) at once. They are often needed if several pieces of data need to be kept in sync.

Multi-object transactions require some way of knwoing which reads and writes belong to the same transaction. 
- In relational databases, that is typically done based on the client's TCP connection to the database server: on any particular connection, everything between a BEGIN TRANSACTION and COMMIT is considered part of the same transaction.
- Many nonrelational databases don't have such a way of grouping operations together.
#### Single-object writes
Storage engines almost universally aim to provide atomicity and isolation at the level of a single object(such as a key-value pair) on one node. 
- Atomicity can be implemented using a log for crash recovery
- Isolation can be implemented by using a lock on an object(allowing only one thread to access an object at any one time)

Some databases also provide more complex atomic operations such as
- *Increment* operation, which removes the need for read-modify-write cycle.
- *Compare-and-set* operation, which allows a write to happen only if the value has not been concurrently modified by someone else.
 
 They are sometimes dubbed as lightweight transactions or ACID, which is a misleading terminology. A transaction is usually understood as a mechanism to group multiple operations on multiple objects into one unit of execution.
 #### The need for multi-object transactions
 Many distributed datastores have abandoned multi-object transactions because:
 - They are difficult to implement across partitions.
 - They can get in the way in some scenarios where very high availability or performance is required.
 
 Multi-object transactions are useful in several cases in which writes to different objects need to be coordinated.
 - When inserting several records that refer to one another, the references(foreign keys) have to be correct and up-to-date, or the data becomes nonsensical.
 - Document databases lacking join functionality encourage denormalization. When denormalized information needs to be updated, we need to update several documents in one go.
 - The secondary indexes are different objects from a transaction's point of view, and need to kept in sync with the data.

 Though such applications can be implemented without transactions, error handling becomes far more complex without atomicity, and and the lack of isoation can cause concurrency problems.
 #### Handling errors and aborts
 A key feature of a transaction is that it can be *aborted and safely retried* if an error occurred.

 Although retrying an aborted trabsaction is a simple and effective error handling mechanism, it isn't perfect.
 - If the transaction actually succeeded, but the network failed while the server tried to acknowledge the successful commit to the client(so the client thinks it failed), then retrying teh transactions causes it to be performed twice--unless we have an application level deduplication mechanism in place.
 - If the error is due to overload, retrying the transaction will make the problem worse, not better. To avoid such feedback cycles we can:
    - Limit the number of retries
    - Use exponential backoff
    - Handle overload-related errors differently than other errors
- It is only worth retrying after transient errors(e.g. due to deadlock, isolation violation, temporary network interruptions, and failover); after a permanent error(e.g. constraint violation) a retry would be pointless.
- If the transaction also has side effects outside of the database, those side effects could still happen even if the transaction is aborted e.g sending an email.
- If the client process fails while retrying, any data it was trying to write to the database is lost.

## Weak Isolation Levels
If two transactions don't touch the same data, they can safely be run in parallel, because neither depends on the other. Concurrency issues(race consitions) only come into play when one transaction reads data that is concurrently modified by anothe rtransaction, or two transactions try ti simultaneously modify the same data.

Concurrency bugs are very hard to find by testing, because such bugs are only triggered when you get unlucky with the timing. Such timing issues might occur very rarely, and are difficult to reproduce.

Concurrency is also very difficult to reason about, especially in a large application, where you don't necessarily know which othe pieces of code are accessing the database.

In theory, isolation should make our life easier by letting us pretend that no concurrency is happening. But serialization has performance cost. In practice, its common for systems to use weaker levels of isolation, which protect against some concurrency issues but not all. Those levels of isolation are much harder to understand and can lead to subtle bugs.

We need to build a good understanding of the concurrency problems that exist, and how to prevent them. Then we can build applications that are reliable and correct, using the tools at our disposal.
### Read Coimmitted
This is the most basic level of transaction isolation. It provides two guarantees:
1. When reading from a database, we will only see the data that has been committed(no *dirty reads*)
2. When writing to te database, we will only overwrite the data that has been committed(no *dirty writes*)
#### No Dirty Reads
There are a few reasons to prevent dirty reads
- A dirty read means that another transaction sees some updates but not others. Seeing the database in partially updated state is confusing for users and could lead other transactions to take incorrect decisions.
- If the database allows dirty reads, that means a transaction may see data that is later rolled back.
#### No Dirty Writes
When two transaction concurrently try to update the same object in the database, we can assume that later write will overwrite the earlier write. But if the earlier write is part of a transactionthat has not yet committed, the later write overrites an uncommitted value.

Transactions running at read committed isolation level must prevent dirty writes, usually by delaying the second write until the first write's transaction is committed.
#### Implementing Read Committed
##### Preventing dirty writes
Most commonly database prevent *dirty writes* by using row-level locks. When transaction wants to modify an object(row or document) it must first acquire a lock on the object. It must then hold this lock until the transaction is committed or aborted. This locking is done automatically by databases in read committed mode(or stronger isolation levels)
##### Preventing dirty reads
The approach of using read locks does not work well in practice, because one long-running write transacion can block all other transactions, even if other transactions only read from the database. This harms the response time of read-only transactions and is bad for operability: a slowdown in one part of the application can have a knock-on effect in a completely different part fo the application, due to waiting for locks.

For that reason databases use the following approach to prevent direty reads.
- For every object that is written, the database holds both the old committed value and the new value set by the transaction that currently holds the write lock.
- While the transaction is ongoing, any other transactions that read the object are simply given the old value.
- only when the new value is committed, do the transactions switch to reading the new value.

### Snapshot Isolation and Repeatable Read
There are many concurrency bugs that may happen while using read-commited isolation.
**Read Skew** 

Read skew is an example of a nonrepeatable read and it is a temporary inconsistency. Some situations cannot tolerate this temporary inconsistency
- *Backups*
    - Taking a backup requires making a copy of the entire database, which may take hours on a large database.
    - During the time the backup process is running, writes will continue to be made to the database. 
    - Thus we could end up with some parts of the backup containing older version of the data, and other parts containing anewer version.
    - If we need to restore from such backup, inconsistencies become permanent.
- *Analytic Queries and Integrity Checks*
    - Analytics queries commonly scan large part of the database
    - Such queries may also be part of a periodic integrity check in order to monitor for data corruption.
    - These queries are likely to return nonsensical results if they observe parts of the database at different points in time.

It is very hard to reason about the meaning of a query if the data it operates on is changing at the same tim ethe query is executing.

**Snapshot isolation** is the most common solution to this problem
- The idea is that each transaction reads from a consistent snapshot of the database--that is, the transaction sees all the data that was committed to the database at the start of a transaction.
- Even if the data is subsequently changed by anotther transaction, each transaction only sees the old data from that particular point in time.
#### Implementing snapshot isolation
Implementations of snapshot isolation typically use write locks to prevent dirty writes. However, reads do not require any locks. From  a performance point of view, a key principle of snapshot isolation is, *readers never block writers and writers never block readers*.

**Multi-version concurrency control(MVCC):** The database must potentially keep different committed versions of the object, because various in-progress transactions may need to see the state pof the database at different points in time. this technique is known as MVCC.
##### Visibility rules for observing a consistent snapshot
An object is visible to a transaction if both of the following conditions are met
- At the time when reader's transaction started, the transaction that created the object had already committed
- The object is not marked for deletion, or if it is, the transaction that marked the object for deletion had not committed when the reader's transacion started.

By never updating values in place but creating a new version every time avalue is changed, the database can provide a consistent snapshot while incurring a small overhead.
#### Indexes and Snapshot Isolation
How do indexes work in multi-version databases?
- One option is to have the index simply points to all versions of an object and require an index query to filter out any object versions that are not visible to the current transaction.
- *Append-only/copy-on-write:
    - Used by CouchDB, Datomic, and LMDB
    - Every write transaction(or batch of transactions) creates a new B-tree root, and a particular root is the consistent snapshot of the database at the point when it was created 
    - this approach requires back ground process for compaction and garbage-collection
#### Repeatable read and Naming Confusion
Even though several databases implement repeatable read, there are big differences in the guarantees they provide
- Oracle calls repeatable read *serializable*
- PostgreSQL and MySQL call it *repeatable read*
- IBM DB2 use repeatable read to refer to *serializability*
### Preventing Lost updates
The read committed and snapshot isolation levels we have disussed so far have been primarily about the guarantees of what a read-only transaction can see in presence of concurrent writes. 
#### The Lost Update Problem
The lost update problem can occur in a *read-modify-write cycle*(if an application reads some value from the database, modifies it and writes back the modified value). If two transactions do this concurrently, one of the modifications can be lost, as teh second write does not contain the first modification.

This pattern occurs in various scenarios
- Incrementing a counter or updating an account balance
- Making a local change to a complex value e.g. adding an element to a list in a json document
- Collaborative editing
#### Solution for the lost update problem
##### Atomic Write Operations
- Many databases provide atomic write operations, which remove the need to perform read-modify-write cycles.
- Not all writes can be expressed in terms of atomic operations. Wherever the writes can be expressed in terms of atomic operations, they are usually the best choice.
- *Cursor stability*: Atomic operations are usually implemented by taking an exclusive lock on the object when it is read, so that no other transaction can read it until the update has been applied.
- Another option is to simply force all atomic operations to be executed on a single thread.
##### Explicit locking
- Application explicitly locks the objects to be updated
- Then the application performs the read-modify-write cycle
- if any transaction concurrently tries to read the same object, it is forced to wait until previous read-modify-write cycle is completed.
- To make this work, we have to carefully think about our application logic. It's easy to forget to add a necessary lock somewhere in the code, and thus introduce a race condition.
##### Automatically detecting lost updates
- Atomic operations and lost updates are ways of preventing lost updates by forcing the read-modify-write cycles to happen sequentially. 
- An alternative is to allow them to execute in parallel and if the transaction manager detects a lost update, abort the transaction and force it to retry its read-modify-write cycle.
- Advantage of this approach is that a database can perform this check efficiently in conjunction with snapshot isolation.
- PostgreSQL's repeatable read, Oracle's serializable and SQL Server's snapshot isolation levels automatically detect when a lost update has occurred and abort the offending transaction.
- MySQL/InnoDB's repeatable read does not detect lost updates
- Lost update detection is a great feature, because it doesn't require application code to use any special database features--you may forget to use a lock or an atomic operation and thus introduce a bug, but lost update detection happens automatically and is less error prone.
##### Compare-and-set
- The purpose of this operation is to avoid lost updates by allowing the update to happen only if the value has not changed since you last read it.
- If the current value does not match what you previously read, the update has no effect, and the read-modify-write cycle must be retried.
- If the database allows reading from an old snapshot, this method may not prevent lost updates.
##### Conflict Resolution and Replication
Locks and compar-and-set assume that there is a single up-to-date copy of the data. However databases with multi-leader or leaderless replication usually allow several writes to happen concurrently on different nodes and replicate them asynchronously, so they cannot guarantee that there is a single up-to-date copy of the data.

A common approach in such replicated databases is to allow concurrent writes to create several conflicting versions of a value(also known as siblings), and to use application code or special data structures to resolve and merge these versions after the fact.

Atomic operations can work well in a replicated context, especially if they are commutative.
- For example, incrementing a counter or adding an element to a set are commutative operations.
- Riak2.0 datatypes prevent lost updates across replicas, by automatically merging the concurrent updates.

The *last write wins* (LWW) conflict resolution method is prone to lost updates. Unfortinately, LWW is default in many replicated databases.
##### Write Skew and Phantoms
###### Characterizing write skew
Write skew is neither a dirty write or a lost update, because the two transactions update two different objects.

We can think of write skew as *generalization of lost update*.
- Write skew can occur if two transactions read the same objects and then update some of those objects(different transactions may update different objects)
- In the special case where different transactions update the same object, we get a dirty write or lost update anomaly(depending on the timing).
###### Preventing write skew
- Automatically preventing write skew requires true serializable isolation
- The second best option is to explicitly lock the rows that transaction depends on.
###### More examples of write skew
- Meeting room booking system
- Multiplayer game
- Claiming a username
- Preventing double-spending
###### Phantoms causing write skew
1. A SELECT query checks whether some requirement is satisfied by searching for rows that match some search condition.
2. Depending on the result of the first query, application decides how to continue(perhaps to go ahead with the operation, or to report error to the user adn abort)
3. If the application decides to go ahead, it makes a write tpo the database and commits the transaction.

The effect of this write changes the precondition of the decision is step 2.If we were to repeat the SELECT query from step 1, we would get a different result, because the write changed the set of rows matching the search condition.

In many cases queries check for *absense* of rows matching some search condition, and the write adds rows matching the same condition. If the query in step 1 doesn't return any rows, SELECT FOR UPDATE can't attach locks to anything.

This effect, where a write in one transaction changes the result of the search query in another transaction is called a phantom.
###### Materializing conflicts
The problem with phantoms is that there is no object to which to attach locks.

*Materializing conflicts* artificially introduces a lock object into the database. It takes a phantom and turns it into a lock conflict ona concrete set of rows trhat exist in a database

*Materializing conflicts should be considered as a last resort if no alternative is possible because:*
- It can be hard and error-prone to fogure out how to materialize conflicts.
- It's ugly to let a concurrency control mechanism leak into the application data model.

## Serializability
Race conditions
- Isolation levels are hard to understand, and inconsistently implemented in different databases
- It is difficult tell by lloking at application code, if it is safe to run at particular isolation level--especially in large application, where we may not be knowing about all the things happening concurrently.
- There are no good tools to detect race conditions
    - Static analysis can help but research techniques have not yet found their way into practical use
    - Testing for concurrency issues is hard, because they are usually non-deterministic

Serializable is usually regarded as the strongest isolation level
- It guarantees that even if transactions execute in parallel, the end result is the same as if they had executed one at a time, serially, without any concurrency.
- Thus, the database guarantees that if the transactions behave correctly when run individually, they behave correctly when run concurrently.
- in other words, database prevents all the race conditions.

Most databases that provide serializability today use one of the three techniques
- Actual Serial execution
- Two Phase Locking(2PL)
- Serializable Snapshot Isolation
### Actual Serial execution
Execute only one transaction at a time, in a serial order, on a single CPU. The resulting isolation is by default serializable.

Single-threaded loops for executing transactions have become feasible due to  two developments:
- RAM became cheap enough that for many use cases it is now feasible to keep the entire acive dataset in memory. When all the data that transaction needs to access is available in memory, transactions can execute much faster than if the data has to be loaded from the disk,
- There was a realization that OLTP transactions are usually small and make only a small number of reads and writes. By contrast, long-running analytic queries are typically read only, and can be run on a consistent snapshot outside the serial execution loop.

This approach of executing transactions is implemented in VoltDB/H-Store, Redis, and Datomic.

A system designed for single-threaded execution can sometimes perform better than a system that supports concurrency, because it can avoid the coordination overhead of locking. However, it's throughput is limited to that of the single CPU core. In order to make the most of that single thread, transactions need to be structured differently from their traditional form.
#### Encapsulating transactions in stored procedures
Almost all OLTP applications keep transactions short by avoiding interactively waiting for a user within the transaction. On the web, this means that:
- The transaction is committed within the same HTTP request
- A transaction does not span multiple requests
- A new HTTP request starts a new transaction

Even then transactions have continued to be executed in an interactive client/server style, one statement at a time
- The application makes a query, reads the result and perhaps makes another query depending on the result of the first query, and so on.
- The queries and results are sent back and forth betwen application code(running on one machine) and the database server(on another machine).
- In this interacyive style of transaction, a lot of time is spent in network communication betweenbetween the application and the database.
- In this kind of database, it's necessary to process multiple transactions concurrenty in order to get reasonable performance.

Systems with single-threaded serial transaction processing do not allow interactive multi-statement transactions.
- The application must submit the entire transaction code to the database ahead of time, as a stored procedure.
- Provided that all data required by a transaction is in memory, the stored procedure can execute very fast, without waiting for any network or disk I/O.
#### Pros and Cons of Stored procedures
Stored procedures have gained bad reputation for various reasons
- Each database vendor has oits own language for stored procedures
- These languages haven't kept up with the devlopments in general-purpose programming languages
    - They look quite ugly and archaic.
    - They lack ecosystem of libraries that you find with most programming languages
- Code running in a database is difficult to manage. Compared to application server it is:
    - It's harder to debug
    - More awkward to keep in version control and deploy 
    - Trickier to test
    - Difficult to integrate with a metrics collection system for monitoring
- A database is usually far more performance sensitive than an application server, because a single database instance is often shared by many application servers. A badly written stored procedure in a database can cause lot more trouble than equivalent badly written code in an application server.

*Modern implementations of stored procedures have abandoned PL/SQL and use existing general purpose programming languages instead: VoltDB uses Java or Groovy, datomic uses Java or Clojure, and Redis uses Lua.*

With stored procedures and in-memory data, executing all transactions on a single thread becomes feasible. As they don't need to wait for I/O and they avoid the overhead of other concurrency control mechanisms, they can achieve quite good throughput on a single thread.
#### Partitioning
For applications with high write throughput, the single-threaded transaction processor can become a serious bottleneck.

In order to scale to multiple CPU cores, and multiple nodes, we can potentially parition our data.
- If we can find a way of partitioning our dataset so that each dataset can read and write data within a single partition, then each partition can have its own transaction processing thread running independently from the others.
- In this case, we can give each CPU core its own partition, which allows your transaction throughput to scale linearly with the number of CPU cores.

However cross-partition transactions(transactions that need to access data from multiple transactions) are vastly slower than single-partition transactions, as they have additional ccordination overhead.

Whether transactions can be single-partition depends very much on the structure of the data used by the application.
- Simple key-value data can often be partitioned quite easily
- But data with secondary-indexes is likely to require lot of cross-partition coordination.
#### Summary of serial execution
Serial execution of transactions has become a viable way of achieving serializable isolation within certain constraints
- Every transaction must be small and fast, because it takes only one slow transaction to stall all transaction processing
- It is limited to use cases where active dataset fits within memory.Rarely accessed data could be potentially moved to disk, but if needed to be accessed in single-threaded transaction, the system would get very slow.
- Write throughput should be low enough to be handled on a single CPU core, or else transactions need to be partitioned without requiring cross-partition coordination.
- Cross-partition transactions are possible, bit there is a hard limit to the extent to which they can be used.
### Two-Phase Locking(2PL)
With two-phase locking:
- Several transactions are comcurrently allowed to read the same object, as long as nobody is writing to it. 
- But as soon as someone wants to write(*modify or delete*) an object, exclusive access is required.

In 2PL, writers don't just block other writers;they also block readers and vice versa.
#### Implementation of two-phase locking.
The blocking of readers and writers is implemented by having a lock on each object in the database. The lock can either be in shared mode or exclusive mode.
- If a transaction wants to read an object, it must first acquire the lock in shared mode.
    - Several transactions are allowed to hold the lock in shared mode simultaneously.
    - But if another transaction already has an exclusive lock on the object, these transactions must wait.
- If the transacion wants to write to an object, it must first acquire the lock in exclusive mode.
    - No other transaction may hold the lock at the same time(whether in shared mode or exclusive mode).
    - If there is any existing lock on the object, that transaction must wait.
- If a transaction first reads and then writes an object, it may upgrade its shared lock to an exclusive lock. The upgrade works the same as getting an exclusive lock directly.
- After a transaction has acquird the lock, it must continue to hold the lock until the end of the transaction(commit or abort). 
    - This is where the name 2PL comes from.
        1. First phase is when locks are acquired
        2. Second phase is when all the locks are released

The database automatically detects any deadlocks between transactions and aborts one of them so that othes can make progress. The aborted transaction needs to be retried by the application.
#### Performance of two-phase locking
Transaction throughput and query response times are significantly worse under two-phase locking than under weak isolation. This is due to:
- Overhead of acquiring and releasing all those locks
- But more importantly due to reduced concurrency

By design, if two concurrent transactions try to do anything that may in any way rsult in a race condition, one transaction has to wait for the other to complete. This is a problem for long running transactions. Even if transactions are kept small, a queue may fprm if several transactions try to read the same object, so transaction may have to wait for several others to complete. 

For this reason, satabase running 2PL can have quite unstable latencies, and they can be very slow at high percentiles if there is contention. It may take just one clow transaction, or a transaction that accesses lot of data and acquires many locks, to cause the rest of teh system to grind to a halt.

Also deadlocks occur far more frequently under 2PL serializable isolation, which can mean significant wasted effort.
#### Predicate locks
A database with serializable isolation must prevent phantoms: If one transaction has queried for existing records based on some search condition, another transaction is not allowed to concurrently insert or update records matching the same condition.

Coceptually, we need a predicate lock. It works similarly to the shared/exclusive lock, but rather than belonging to a particular object, it belomgs to all objects that match the search condition.

A predicate lock restricts access as follows:
- If transaction A wants to read objects matching some search condition, it must acquire a shared-mode predicate lock on the conditions of the query. If another transaction B has an exclusive lock on any object matching those conditions, A must wait until B releases its lock before it is allowed to make its query.
- If transaction A wants to insert, update, or delete any object, it must first check whether either the old or new value matches any existing predicate lock. If there is a matching predicate lock held by transaction B, then A must wait until B has committed or aborted before it can continue.

The key idea here is that the predicate locks apply even to objects that do not exist yet in the database, but which might be added in future(*phantoms*).

If 2PL includes predicate locks, the database prevents all kinds of write skew and other race conditions, and so its isolation becomes serializable.
#### Index-range locks
Predicate locks do not perform well: if there are many locks by active transactions, checking for matching locks becomes time-consuming. So, most databases with 2PL implement *index-range locking*(aka *next-key locking*)which is an approximation of predicate locking.

It's safe to simplify a predicate by making it match a greater set of objects. Any write that matches the original predicate will definitely also match the approximation.

The database can simply attach a shared lock to the index entries matching the approximation of search condition. Now if another transaction wants to insert, update or delete a record matching the approximation, it will have to update the same part of the index. In the process of doing so, it will encounter the shared lock and will be forced to wait until the lock is released.

This provides effective protection against phantoms and write skew
- Index range locks are not as precise as predicate locks(they may lock a bigger range of objects than is strictly necessary to maintain serializability). 
- But since they have much lower overheads, they are a good compromise.
### Serializable Snapshot Isolation(SSI)
*Serializable snapshot isolation* provides full serializability, but has only a small performance penalty compared to snapshot isolation.

SSI is used in single-node databases(PostgreSQL) as well as distributed databases(FoundationDB).
### Pessimistic vs Optimistic Concurrency Control
*2PL is  a pessimistic cpncurrency control mechanism:* it is based on the principle that if anything might possibly go wrong(as indicated by a lock held by another transaction), it's better to wait until the sitiuation is safe again before doing anything.

*Serial execution is pessimistic to teh extreme:* It is essentially equivalent to each transaction having an exclusive lock on teh entire database(or one partition of the database) for the duration of the transaction. We compensate for the pessimism by making each transaction very fast to execute, so it only needs to hold the lock for a very small time.

By contrast *SSI is an optimistic concurrency control technique*
- Instead of blocking when something potentially dangerous happens, transactions continue anyway, in the hope that everything will turn out all right. 
- When a transaction wants to commit, the database checks whether anything bad happened(i.e. whether isolation was violated); if so, tha transaction is aborted, and has to be retried.
- Only transactions that executed serializably are allowed to commit.

Optimistic concurrency control performs badly if there is high contention(many transactions trying to access the same objects), as this leads to high proportion of transactions needing to abort. 

If there is enough spare capacity, and if contention between transactions is not too high, optimistic concurrency control techniques tend to perform better than pessimistic ones. Contention can be reduced with commutative atomic operations.

SSI is based on *snapshot isolation*
- All reads within a transaction are made from a consistent snapshot of the database. This is the main difference to the earlier optimistic concurrency control techniques.
- On top of snapshot isolation, SSI adds SSI adds an algorithm for detecting serialization conflicts among writes and determining which transactions to abort.
#### Decisions based on an outdated premise
Many transactions have following recurring pattern
- Read some data from the database
- Examine the results of the query
- Decide to take some action(write), based on the result that it software

In this case, transaction is taking an action based on a premise(the fact that was true at the begining of the transaction). Under snapshot isolation, by the time the transaction commits, the premise may no longer be true(data/facts might have changed).

In order to provide serializable isolation, the database must detect situations in which a database may have acted on an outdated premise and abort the transaction in that case.

Database can detect if the query result might have changed for two cases
- Detecting reads of a stale MVCC object version(uncommitted writes occurred before the read)
- Detecting writes that affect prior reads(the write occurs after the read)
##### Detecting stale MVCC reads
- The databse needs to track whena transaction ignores another transactions writes due to MVCC visibility rules.
- When the transaction wants to commit, the database checks if any of the ignored writes have now been committed. If so, the transaction must be aborted.
- By avoiding unnecessary aborts, SSI preserves snapshot isolation's support for long-running reads from a consistent snapshot.
##### Detecting writes that affect prior reads
- When the transaction writes to the database, it must look in the indexes for any other transactions that have recently read the affected data.
- The database notifies these transactions that the data they read may no longer be up to date.
#### Performance of SSI
Many enginering details affect how well an algorithm works in practice
- *Granularity at which transactions reads and writes are tracked*
    - If the database keeps track of each transaction's activity in great detail, it can be precise about which transactions need to abort,but the bookkeeping overhead can become significant.
    - Less detailed tracking is faster, but may result in more transactions being aborted than strictly necessary.
- In some cases it is ok for a transaction to read information overwritten by another transaction: depending on what else happened, it sometimes possible to prove that the result of the execution is nevertheless serializable.
- Compared to 2PL, the big advantage of SSI is that one transactuon doesn't need to block waiting for locks held by another transaction.
    - Writers don't block readers and vice versa
    - This design principle makes query latency much more predictable and less variable.
    - Read only queries can run on a consistent =snapshots without requiring any locks, which is appealing for read-heavy transactions
- Compared to serial execution SSI is not limited to the throughput of the single CPU core
- The rate of aborts significantly affects the performance of SSI
    - SSI requires that read-write transactions be fairly short to reduce the probability of an abort.
    - SSI is probably less sensitive to slow transactions than 2PL or serial execution. 












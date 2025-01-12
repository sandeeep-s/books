# Stream Processing
In reality, a lot of data is unbounded, so the dataset is never complete in any meaningful way. The batch processes must artificially divide data into chunks of fixed duration. So the changes in input reflect are only reflected in output a day later.

The idea behind stream processing is to run processing
- Frequently--say processing a second's worth of data at the end of every second.
- Or Contnuously--abandoning fixed time slices completly and simply processing the events as they happen.

*Stream*, in general, refers to data that is incrementally made available over time. The concept appears in many places:
- The stdin and stdout of Unix 
- Programming languages(lazy lists)
- Filesystem APIs
- TCP connections
- Delivering audio and video over the internet
- Others

*Event streams* as a data management mechanism are the unbounded, incrementally processed counterpart to the batch data.

## Transmitting Event Streams
*Event* is a small, self-contained, immutable object containing the details of something that happened over time. 
- Few examples of, the thing that happened might be:
    - An action that a user took, such as viewing a page or making a purchase
    - A periodic measurement from a temperature sensor
    - A CPU utilization metric
- An event usually contains a timestamp indicating when it happened according to the time-of-day clock.
- An event maybe encoded as a text string, or JSON, or perhaps in some binary form.
    - This encoding allows to store an event by appending it to a file, inserting into a relational database, or writing it to a document database.
    - It also allows to send event over the network to be procesed by another node.
- An event is generated once by a *producer* (aka *publisher* or *sender*). and then potentially processed by multiple *consumers*(aka *subscribers* or *recipients*).
- In a streaming system, related events are usually grouped into a *topic* or *stream*.

In principle, a producer could write events to a datastore, and each consumer polls the datastore to check for any new events. However polling is expensive and adds overheads when done too often. It is better for consumers to be notified when new events occur.

Databases have traditionally not supported this kind of notification mechanism very well: triggers are very limited in what they can do and have been somewhat of an afterthought.
### Messaging Systems
A common approach for notifying consumers about new events is to use a *messaging system*: a producer sends a message containing the event, which is then pushed to the consumers. 

A messaging system allows muliple producer nodes to send messages to the same topic and allows multiple consumer nodes to receive messages in a topic.

Within this publish/subscribe model, different systems take broad range of approaches primarily based on following two questions:
1. *What happens if producers send messages faster than consumers can process them?*

    Broadly speaking there are three options:
    - The system can drop messages
    - The system can buffer messages in a queue 
    - Apply backpressure(aka flow control i.e. blocking the producer from sending more messages)

    If messages are buffered in a queue, it is important to understand what happens when the queue grows.
    - Does it crash if the queue no longer fits in memory?
    - Or does it write messages to disk? In this case, how does the disk access affect the performance of the messaging system?
2. *What happens if nodes crash or temporarily go offline--are any messages lost?*

    Durability may require some combination of writing to disk and/or replication, which has a cost. If we can afford to lose messages, we can probably get higher throughput and lower latency on the same hardware. Whether message loss is acceptable depends totally on the application.
#### Direct messaging from producers to consumers
A number of messaging systems use direct network communication between producers and consumers
- UDP multicast is widely used in the financial industry for streams such as stock market feeds where low latency is important.
- Brokerless messaging libraries such as ZeroMQ and nanomsg
- StatsD and Brubeck
- Webhooks: producers make direct HTTP or RPC calls to push messages to the consumer

Although these direct messaging systems work well in the situations for which they are designed, they generally require the application code to be aware of the possibility of the message loss. 
- The faults they can tolerate are quite limited. 
- They generally assume that producers and consumers are always online.
#### Message brokers(aka Message queues)
Message broker is a kind of database optimized to handle message streams. 
- It runs as a server, with producers and consumers connecting to it as clients.
- Producers write messages to the clients and client receive the messages by reading them from the broker.
- Centralization of data in the broker allows more easily tolerating the clients that come and go(connect, disconnect and crash)
- Some message brokers keep messages only in memory, while others write them to disk(often configurable)
- They generally allow unbounded queuing.
- Consumers are generally *asynchronous*.
#### Message brokers compared to databases
- Databases usually keep data until it is explicitly deleted. Most message brokers automatically delete a message on successful delivery to consumers.
- Most message brokers assume there working set is fairly small i.e. queues are short. Too much buffering may degrade performance and throughput.
- Databases often support secodary indexes and some way of searching for data. Message brokers often support some way of subscribing to a subset of topics matching some pattern. Both are essentially ways for clients to select the portion of the data they wan to know about.
- When querying a database, the result is typically based on a point-in-time snapshot of the data. Message brokers do not support arbitrary queries, but they notify clients when the data changes.

This is teh traditional view of message brokers, which is encapsulated in the standards like JMS and AMQP and implemented in software like RabbitMQ, ActiveMQ,HornetQ, Qpid, TIBCO Enterprise Message Service, IBM MQ, Azure Service Bus, Google Cloud Pub/Sub.
#### Multiple Consumers
When multiple consumers read messages from the same topic, two main patterns of mesaging are used

*Load Balancing*
- Each message is delivered to one of the consumers.
- Consumers share the work of processing messages in the topic.
- Useful when messages are expensive to process, and so we want to be able to add consumers to parallelize the processing.

*Fan-out*
- Each message is delivered to all of the consumers
- Allows several independent consumers to each "tune-in" to the same broadcast of messages without affecting each other.

The two patterns can be combined: for example, two seperate groups of consumers may each subscribe to a topic, such that each group collectively receives all messages, but within each group only one node receives the message.
#### Acknowledgements and redelivery
In order to ensure that message is never lost, message brokers use *acknowledgements*:
- A client must explicitly tell the broker that it has finished processing the message so that the broker can remove it from the queue.
- If the connection to the client is closed or times out without the broker receian acknowledgement, it assumes that the message was not processed, and therefore it delivers the message again to another consumer.
- When combined with load balancing, the redelivery behavior can result in messages being delivered in different order.

### Partitioned logs
Message brokers are built around trasient messaging mindset. They do not store state permanently. Conversely a database or file  is expected to store data permanently.

A key feature of batch processing is that we can run them repeatedly, experimenting with the processing steps, without risk of damaging the input(since the input is read-only). This is not the case with AMQP, JMS stale of messaging: receiving the message is desructive if the acknowledgement causes the message to be deleted from the broker. Input is not immutable. We cannot run the same consumer again and expect to get the same result. A new consumer typically only starts receiving messages sent after the time it was registered. Contrasticly for databases and files, new clients can read data written arbitrarily far in the past.

#### Log-based message brokers
The idea behind *log-based message brokers* is to combine the durable storage approach of the databases with the low-latency notification facilities of messaging.
##### Using logs for message storage
A log is simply an append-only sequence of records on disk. The same structure can be used to implement a message broker:
- A producer sends message by appending it to the end of the log.
- A consumer receives messages by reading the log sequentially.
- If the consumer reaches end of the log, it waits for a notification that a new message has been appended.

The log can be partitioned for higher throughput
- Different partitions can then be hosted on different machines, making each partition a separate log that can be read and written independently from other partitions.
- A topic can then be defined as a group of partitions that all carry the messages of the same type.
- Within each partition, the broker assigns a monotonically increasing sequence number, or *offset*, to every message.
- Examples: 
    - Apache Kafka
    - Amazon kinesis Streams
    - Twitter's Distributed log
    - Google Cloud Pub/Sub(with JMS style API rather than a log abstraction)
- These message brokers write all messages to disk. They are able to achieve throughput of millions of messages per second by partitioning across multiple machines, and fault tolerance by replicating messages.
##### Logs compared to traditional messaging
The log based approach trivially supports fan-out messaging, because several consumers can independently read the log without affecting each other--reading a message does not delete it from the log.

*Load Balancing:*
- The broker can assign entire partitions to nodes in consumer groups
- The node then reads all the messages in the assigned partion sequentially, in a single-threaded manner.
- This coarse grained load-balancing approach has some downsides
    - The number of nodes sharing the tpoic load can be at most the number of log partitions in the topic.
    - If a single message is slow to process, it holds up the processing of subsequent messages in the parition.

JMS/AMQP style of messaging is preferable in cases where:
- Messages may be expensive to process and we want to prallelize processing on message-by-message basis
- Message ordering is not important

The log based approch works well in situations:
- With high message throughput
- Where each message is fast to process
- Message ordering is important

*Message ordering*: Since paritioned logs typically preserve message ordering only within a single partition, all messages that need to be ordered consistently need to be routed to the same partition.
#####  Consumer offsets
Since paritions are consumed sequentially, the log based broker does not need to track acknowledgements for every message--it only needs to periodically record the consumer offsets.

The reduced bookkeeping overhead and the opportunities for batching and pipelining in this apporach help increase the throughput of log-based systems.

This offset is similar to *log sequence number* in single leader replication: the message broker behaves like a leader database, and the consumer like a follower.

In case a consumer fails, another node is assigned the partition and starts reading mesages from the last recorded offset.
##### Disk space usage
If we only ever append to the log, we will eventually run out of disk space.

To reclaim disk space, log is divided into segements, and from time to time old segments are deleted or moved to archive.
- Effectively, the log implements a bounded-size buffer that discards old messages when it gets full.
- It is also known as *circular buffer* or *ring buffer*.
- Since the buffer is on the disk, it can be quite large. A log can typically keep a buffer of several days or even weeks worth of messages.

Irrespective of how long we retain messages, the throughput remains more or less constant, since every message is writtent to disk anyway. For messaging systems that keep messages in memory by default and only write them to the disk if the queue grows too large, the throughput depends on the amount of history retained.
##### When consumers cannot keep up with producers
The log-based approach is a form of buffering with a large but fixed-size buffer(limited by the available disk space). The broker effectively drops messages that go back further than what the size of the buffer can accomodate.

*Operational advantage of logs:*
- Logs have an operational advantage: we can experimentally consume production logs for development, testing, or debugging purposes, without having to worry too much about disrupting production services. When a consumer shuts down or crashes, it stops consuming resources--the only thing that remains is it's offset.
- With traditional messaging systems, we need to be careful to delete any queues whose consumers have been shutdown--otherwise they continue unnecessarily to accumulate messages and take away memory from consumers that are still active.
##### Replaying old messages
In log-based message brokers, consuming messages is a read only operation that does not change the log. Only side-effect is that the consumer offset moves forward.But the offset is under consumer's control, which can be easily manipulated (moved back and forth) as necessary. 

This aspect makes log-based message processing more like batch processes, where derived data is clearly seperated from the input data through a *repeatable transformation process*. It allows more experimentation and easier recovery from errors and bugs, making it a good tool for integrating dataflows within an organization.

## Databases and Streams
Log-based message brokers have been successful in taking ideas from databases and applying them to messaging. We can also take ideas from messaging and streams, and apply them to databases.

*Write to a database* is an event that can be captured, stored and processed. This observation suggests that connection between databases and logs is fundamental.
- Replication log is a stream of database write events, produced by the leader as it processes transactions. The events in the replication log describe the data changes that occurred.
- *State machine replcation* is just another case of event streams.

We can solve problems that arise in heterogenous data systems by bring ideas from event streams to databases.
### Keeping Systems in Sync
Most nontrivial applications need to combine several different technologies to satisfy their data storage, querying and processing needs: For example:
- OLTP database to serve user requests
- A cache to speed up common requests
- A full-text index to handle search queries
- Data warehouse for analytics

Each of these has it's own copy of the data, stored in its own representation and optimized for its own purposes. The data in these systems can be kept in sync using *batch processes* or *dual writes*.


In *dual writes*, the application codes explicitly writes to each system when the data changes. Dual writes have some serious problems like race conditions(concurrency problem) and non-atomic beavior(fault-tolerance problem).

The situation would be better if one among the database or search index was the leader--for example, the database--and if we could make the search index a follower of the database.
### Change Data Capture
Change data capture(CDC), is the process of observing all the data changes written to a database and extracting them in a form in which they can be replicated to other systems. 
- CDC is especially interesting if the changes are made available as a stream, immediately as they are written. 
#### Implementing Change Data Capture
CDC is a mechanism for ensuring that the changes made to the system of record are also reflected in the derived data systems, so that dervived data systems have the accurate copy of the data.
- If the log of changes is made in the same order we can expect the dta in derived data systems to match the data in the system of record.
- CDC essentially makes one database the leader and turns others into followers.

A log-based message broker is well suited for transporting the change events from the source database to the dervived systems, since it preserves the ordering of messages.

Database triggers can be used to implement CDC by registring triggers that observe all changes to data tables and add corresponding entries to a changelog table. But they tend to be fragile and have significant performance overheads.

Parsing the replication logs can be a more robust approach, but it comes with challenges, such as handling schema changes. Examples:
- Facebook's Wormhole, LinkedIn's Databus, and Yahoo's Sherpa do it at large scale
- BottleWater for PostgreSQL
- Maxwell and Debzium for MySQL
- Mongoriver for MongoDB
- GoldenGate for Oracle
- Kafka Connect offers CDC connectors for various databases

CDC is usually asynchronous
- This has operational advantage that adding a slow consumer does not affect the system of record too much
- But teh downside is that all the issues of replication lag apply
##### Initial snapshot
If we don't have the entire log history of the database, we need to start with a consistent snapshot.

The snapshot must corresspond to a known offset in the change log, so that we know at which point to start applying changes once the snapshot has been processed. Some CDC tools integrate this snapshot facility while others leave it as a manual operation.
##### Log compaction
In log compaction, the storage engine periodically looks for records with the same key, throws away any duplicates, and only keeps the most recent update for each key.Log compaction allows us to keep limited amount of log history as the disk spcae required depends only on the current contents of the databases. 

If the CDC system is setup in such a way that every change has a primary key, and every update to that key replaces the previous value for that key,  then it's sufficient to keep just the most recent write for that particular key. Now whenever we want to rebuild a dervived data system, we can start a new consumer from offset 0 of the log-compacted topic, and sequentially scan over all messages in the log. The log is guaranteed to contain the most recent write for evry key in the database(and maybe some old values).

This log compaction feature is supported by Apache Kafka. It allows the message broker to be utilized for durable storage, and not just transient messaging.
##### API support for change streams
Increasingly, databases are begining to support change streams as first-class interface. 
- RethinkDB allows queries to subscribe to notifications, when the results of a query change.
- Firebase and CouchDB provide data synchronization based on a change feed that is also made available to applications.
- Meteor uses MongoDB oplog to subscribe to data changes and update the UI
- VoltDB allows transactions to continuously export data from a database in the form of a stream.
- Kafka Connect integrates CDC tools for a wide range of database systems with Kafka.

### Event Sourcing
Similar to CDC, event sourcing involves storing all changes to the application state as a log of change events. But it applies the idea ata different level of abstraction.
- In CDC:
    - The application uses the database in a mutable way, updating and deleting records at will. 
    - The log of changes is extracted from the database at a low level. 
    - The application writing to the database does not need to be aware that CDC is occuring.
- In event sourcing: 
    - The application logic is explicitly built on the basis of immutable events that are written to an event log.
    - The event store is append-only, and updates or deletes are discouraged or prohibited.
    - Events are designed to reflect things that happended at the application level, rather than low-level state changes.

Event sourcing is a powerful technique for data modeling
- From an application's point of view, it is more meaningful to record the user's actions as immutable events, rather than recording the effects of those actions on a mutable database.
- Event sourcing advantages:
    - Makes it easier to evolve system over time.
    - Helps with debugging by making it easier to understand after the fact why something happened.
    - Guards against application bugs.
    - Storing as events, clearly expresses intent of user actions in a neutral fashion. On the contrary, recording side effects approach can embed a lot of assumptions about the way the data is later going to be used. If new application feature is introduced, the event sourcing approach allows the new side effects to be easily chained off the existing events.
 
 Event sourcing is similar to the *chronicle data model*, and has some similarities to the *fact table in a star schema*.

Event sourcing approach is independent of any particular tool. It can be implemented using a specialised datastore like Event Store, a conventional database, or a log based message broker.
#### Deriving current state from the event log
Applications that use event sourcing need to take the log of events(representing the data *written* to the system) and transform it into application state that is suitable for showing it to a user(the way in which the data is *read* from the system). This transformation should be detrministic.

Like with CDC, replaying the event log allows reconstructing the current state of the system. 

However, log compaction needs to be handled differently
- A CDC event typically contains the entire new version of the record.
    - So the current valiue for the primary key is entirely determined by the most recent event for that primary key
    - The log compaction can discard previous events for the same key.
- Event sourcing models events at higher level.
    - An event in this approach typically represents the intent of user action, not the mechanics of state update that happened as a result of the action.
    - In event sourcing, later events do not override prior events, and so we need a full history of events to rconstruct the final state.
    - Log compaction is not possible in the same way as CDC.

Event sourcing applications typically use snapshots of current state to avoid repeatedly reprocessing the entire log. But this is just a  performance optimization to spped up reads and recover from crashes. The intent of an event sourcing is that the system is able to store all raw events forever and reporcess the full event log whenever required.
#### Commands and events
The event sourcing philosophy is careful to distinguish betwen commands and events.
- *Command:* When request from a user first arrives, it is initially a command:The application must first validate if it can execute the command.
- *Event:* If the validation is successful and the command is accepted, it becomes an event, which is durable and immutable. At the point when event is generated, it becomes a fact.

A consumer of an event stream is not allowed to reject an eventAny validation of the command needs to happen synchronously, before it becomes an event. 

Alternatively a request may be split into two events: a tentative event and final confirmation event. This split allows the validation to take place in an asynchronous process.
### State, Streams and Immutability
The principle of immutable input makes batch processing, event sourcing, and CDC powerful.

We normally think of databases as storing the most current state of the application--this representation is optimized for reads, and usually the most convenient for serving queries. Databases support inserting, updating and deleting data.

The mutable state and append-only log of immutable events are two sides of the same coin. 
- The nature of the state is that it changes. It is the result of events that mutated it over time.
- The log of all changes, the *changelog*, represents the evolution of state over time.
- Mathematically(though the analogy has limitation):
    - Application state is what we get, on integrating event stream over time.
    - Change stream is what we get, when we differentiate the state by time.

If we consider the log of events to be our system of record, and any mutable state as being derived from it, it becomes easier to reason about the flow of data through a system. As Pat Helland puts it:

        Transaction logs record all the changes made to the database. High-speed appends are the only way to change the log. From this perspective, the contents of teh database hold a caching of the latest record values in the log. The truth is the log. The database is a cache of a subset of the log. That cached subset happens to be the latest value of each record and index value from the log.

Log compaction is one way of bridging the distinction between log and database state.
#### Advantages of immutable events
- If we accidently deploy a buggy codethat writes bad data to a database, recovery is much harder if the code is able to destructively overwrite the data.  With an append-only log of immutable events, it is much easier to diagnose what happened and recover from the problem.
- Immutable events also capture more information than just the curent state e.g. that a customer added and then deleted item from a cart.
#### Deriving several views from the same event log
By seperating mutable state from immutable event log, we can dervie many different read-oriented representations from the same log of events. This works just like having multiple consumers of a stream. For example:
- The analytic database Druid ingests directly from Kafa using this approach
- Pistachio is a distributed key-value store that uses Kafka as a commit log
- Kafka Connect sinks can export data from Kafka to various databases and indexes.

Having an explicit translation step from an event log to a database makes it easier to evolve our application over time:if we want to introduce a new feature that presents the existing data in some new way, we can use the event-log to build a separate read-optimized view for the new feature, and run it alongside existing systems without having to modify them. Running old and new systems side by side is often easier than performing a complicated schema migration in an existing system. Once the old system is no longer needed, we can simply shut it down and reclaim its resources.

*Command query responsibility segregation*(CQRS):
- Many of the complexities of schema design, indexes and storage engine are a result of wanting to support certain query and access patterns.
- The traditional approach to schema design is based on the fallacy that the data must be written in the same form as it will be queried.
- Debates about normalization and denormalization become largely irrelevant if we can translate data from a write-optimized event log to a read-optimized application state: it is entirely reasonable to denormalize the data in read-optimized views, as the translation process provides us with a mechanism to keep it consistent with the event log.
- We gain a lot of flexibility by:
    -  Separating the the form in which data is written from the form in which the data is read
    - And supporting several different read views
#### Concurreny control
Due to asynchronous nature of event sourcing and CDC, a user may write to the log, but may not immediately see the data in the log-derived view. This could be solved in few ways:
- Update the read view synchronously with appending event to the log. This requires a transaction to combine both writes in an atomic unit. This can be achieved by:
    - Either keeping the event log and read view in teh same storage system
    - Or use distributed transactions
- Alternatively we could use approach discussed in "Implementing linearizable storage using total order broadcast"

Deriving the current state from the event log simplifies some aspects of concurreny control.
- With event sourcing, the event can be designed to be a self-contained description of a user action. The user action then requires only a single write in one place----namely appending event to the log--which is easy to make atomic.
- If the event log and application state are partitioned in the same way, then a single-threaded log consumer needs no concurreny control for writes--by construction it processes only a single event ata time. The log removes the nondeterminism of concurrency by defining a serial order of execution in a partition.
- If an event touches more than one state partition, a bit more work is required.
#### Limitations of Immutability
The feasibility of keeping the immutable history of all changes depends on the amount of churn in the data.
- Workloads that mostly add data but rarel update or delete are easy to make immutable.
- Other workloads have high rate of updates and delete on a comparatively small dataset; in these cases:
    - The immutable history may go prohibitively large
    - Fragmentation may become an issues
    - Performance of compaction and garbage collection becomes crucial for operational robustness.

*Deletion of data:*
- There may be a need to delete data for administrative or legal reasons. It is not siufficient to just append an event to indicate deletion of the data--we actually need to reqrite history and pretend that data was never written in teh first place. For example:
    - *Datomic* calls this feature excision
    - Fossile version control has a similar concept called *shunning*
- Truely deleting data is surprisingly hard:
    - Copies of data can live in may places: e.g. storage engines, filesystems, and SSDs often write to a new location rather than overwriting in  place.
    - Backups are often immutable to prevent accidental deletion or prevention.
    - Deletion is more a matter of "making it harder to retrive the data" than actually "making it impossible to retrieve the data".

## Processing Streams
Streams have three major aspects:
- Where the streams *originate* from (user activity events, sensors, and writes to a database)
- How streams are *transported* (through direct messaging, message brokers, and event logs)
- What to do with streams when we have it--namely *processing it*

We have broadly three options for processing the data
1. Take the data in events, and wite it to a database, cache, search index or similar storage system from where it can be queried by other clients.
2. Push the events to users in some way, in which case a human is the ultimate consumer of the stream. For example:
    - Sending email alerts or push notifications
    - Streaming the events to a real time dashboard where they are visualized
3. Process one or more input streams to produce one or more output streams. Streams may go through a pipeline consisting of several such processing stages before they eventually end up at an output.

A piece of code that processes streams is called an *operator* or a *job*.

Many aspects of stream processing are similar to Unix processes, MapReduce jobs and database engines
- A stream processor reads its input in a read-only fashion and writes it's output to a different location in an append-only fashion.
- Stream procesors use similar patterns for partitioning and parallelization
- Basic mapping operations like transforming and filtering records also work the same.

The one crucial difference from batch processing is that a stream never ends(unbounded dataset):
- Sorting does not make sense with unbounded datasets. So sort-merge joins can't be used.
- Fault-tolerance mechanisms must change. Restarting processing from begining is mostly not feasible.
### Uses of Stream Processing
Stream processing has long been used for monitoring purposes, where organization wants to be alerted if certain things happen e.g. fraud detection systems, trading systems, manufacturing systems, military and intelligence systems etc. These kind sof applications require quite sophisticated pattern matching and correlations.

Other uses of streams that have emerged over time are
#### Complex event processing
*Complex event processing*(CEP) is an approach for analyzing event streamsgeared toward the kind of applications that require searching for certain event patterns.
- CEP allows us to specify rules to search for certain patterns of events in a stream.
- CEP describes event patterns to be searched, using a high-level decalarative query language like SQL, or GUI.
- These queries are submitted to a processing engine which internally maintains a state machine that performs the required matching.
- Once a match is found it emits a complex event with details of the event pattern that was detected.

CEP engines reverse the relationship between queries and data compared to normal databases.
- Queries are stored long-term(as against databases which considers them transient).
- Events from the input stream continually flow past them in search of a query that matches the event pattern.

Implementations of CEP:
- Esper
- IBM Infosphere Streams
- Apama
- TIBCO StreamBase
- SQLstream
- Samza
#### Stream analytics
Stream analytics is oriented towards aggregations and statistical metrics over a large number of events. For example:
- Measuring the rate of some type of event(how often it occurs per time interval)
- Calculating the rolling average of a value over some time period
- Comparing current statistics to previous time intevals(e.g. to detect trends or to alert on metrics that are unusually  high or low compared to the same time last week)

Such statistics are usually computed over a fixed time interval--for example, we might want to know the average number of queries per second to a service over the last 5 minutes, and their 99th percentile response time during that period.
- Avergaing over a few minutes smoothes out irrelevant fuctuations from one second to the next, while still giving us a timely picture of any changes in the traffic pattern.
- The time interval over which we aggregate is known as *window*.

Streams sometimes use probabilistic algorithms like:
- Bloom filters for set membership
- HyperLogLog for cardinality estimation
- Various percentile estimation algorithms

Many open source distributed stream processing algorithms have been designed with analytics in mind:
- Apache Storm
- Spark Streaming
- Flink
- Concord
- Samza
- Kafka Streams
- Google Cloud Dataflow
- Azure Stream Analytics
#### Maintaining materialized views
*Marerialized views* are an alternative view derived onto some dataset so that we can query it efficiently and are updated whenever the underlying data changes.

Dervied data systems such as caches, indexes, data warehouses, application state in event sourcing systems are specific example of materialized view.

Building the materialized view potentially requires all events over an arbitrary time period, aprt from any obsolete events that may be discarded by log compaction.

Samza and Kafka streams specifically support this kind of usage based on Kafka's support for log compaction. 
#### Search on streams
Search for individual events based on a complex search criteria, such as full-text search queries. For example, media monitoring services, real estate notification configuration based on some criteria etc.

In contrast to conventional search engines, the queries are stored, and the documents are run past the queries.

One option for implementing this kind of stream search is the percolator feature of ElasticSearch.
#### Message passing and RPC
We do not usually consider message passing systems(e.g. actor model) as stream processors.
- Actor model is primarily a means of managing concurrency and distributed execution of communicating modules, whereas stream processing is primarily a data management technique.
- Communication between actors is ofthen ephemeral and one-to-one. Event logs are durable and multi-subscriber.
- Actors can communicate in arbitrary ways(including cyclic request/response patterns), but stream processors are usually set up in acyclic pipelines where  every stream is an out of a particular job, and derived from a well-defined set of input streams.

That said, there is some crossover between RPC-like systems and stream processing.
- Apache Storm has afeature called distributed RPC
    - It allows the user queries to be farmed out to a set of nodes that also process event streams.
    - These queries are then interleaved with events from the input streams, and results can be aggregated and send back to the user.

- It is also possible to process stremas using actor frameworks.
### Reasoning About Time
Batch processes use timestamps embedded in events for time windowing and not the system clock.

On the other hand, many stream processing frameworks use local system clock on the procesing machine(*the processing time*) to determine windowing.
- This approach has advantage of being simple.
- Also the approach is reasonable if the delay between event creation and event processing is negligebly short.
- However this breaks down if there is a significant processing lag--i.e. the processing may happen noticeably later than the time at whih the event actually occurred.
#### Event time verses processing time
There are many reasons why processing may be delayed
- Queuing
 - Network faults
 - Performance issues leading to contention in the message broker or processor.
 - A restart of the stream consumer
 - Reprocessing of past events: while recovering from a fault or after fixing a bug in the code.

 Message delays can also lead to unpredictable ordering of messages.

 Stream processing algorithms need to be specifically written to accomodate such timing and ordering issues.

 Confusing event time and processing time leads to bad data.
 #### Knowing when you're ready
 A tricky problem when defining windows in terms of event time is that: we can never be sure when you have received all the events for a particular window, or whether there are some events still to come.

 We need to handle *straggler* events that arrive after the window has been declared complete. Broadly we have two options:
 1. Ignore the straggler events as they are probably a small percentage of events in normal circumstances. Track the number of dropped events as metric and alert if we start dropping significant no.of events.
 2. Publish a *correction*, an updated value for the window with stragglers included. We may also need to retract previous output.
 #### Whose clock are you using, anyway?
 Assigning timestamps to events is even more difficult when events can be buffered at several points in the system. 
 
 For example, a mobile app that reports events for usage metrics  to a server. 
 - The app mmay be used while device is offline thus buffering events locally on the device.
 - It will send the events to server when the internet connection is next available.
- Mobile devices local clock may be accidentally or deliberately set to wrong time.

To adjust for incorrect device clocks, one approach is to log three timestamps
1. The time at which the event occurred, as per the device clock.
2. The time at which the event was sent to the server, as per the device clock.
3. the time at which the event was received by the server, according to the server clock.

The offset between the device time and server time is the difference between 2nd and 3rd timestamps. We can use the offset to determine the true time at which the event occurred.

Batch processing also suffers from the same problem, though it is more visible in stream processing, where we are more aware of passage of time.
#### Type of windows
##### Tumbling window
The tumbling widow has a fixed length, and every event belongs to a single window.
##### Hopping window
A hopping window also has a fixed length, but allows windows to overlap in order to allow some smoothing.
##### Sliding window
A sliding window contains all events that occur within a time interval of each other.
##### Session window
A session window has no fixed duration. It is defined by grouping together all the events for the same user that occur closely together in time. The window ends when the user has been inactive for some time. Sessionizatio is a common requirement for website analytics.

### Stream Joins
Stream processing generalizes data pipelines to incremental processing of unbounded datasets and has a need for joins on streams. The fact that the events can appear anytime on a stream makes joins on streams more challenging than in batch jobs.

Let's distinguish three different types of joins.
#### Stream-stream joins(Window join)
Joins together events from 2 streams e.g. joing the events for search action and click action to to calculate the click-through rate for each URL in the search results
- To impelment this type of join, the stream processor needs to maintain state: for example, all the events that occurred in the last hour, indexed by a join key.
- Whenver events occur they are added to the appropriate indexes and the stream processor also checks the other index to see if another event for the same join key has already arrived.
- If there is a matching event, we can emit an event representing the join between those events.
- If one of the event expires without seeing matching event, we still emit a join event reprsenting absence of one of the event.
#### Stream-table joins(Stream enrichment)
*Enrich* events from a stream with information from the database e.g. augmenting events i user activity stream with profile information of the user based on UserID.
- To perform this join, the stream processor needs to look at one event at a time, look up the record corresspdoning to the event in the database, and add the information from database to the event.
- The database lookup could be implemented by:
    - Querying the remote database 
        - Slow as it involves network round-trip
        - Risk overloading teh database
    - Loading the copy of remote database into the stream processor(similar to hash joins)
        - Can be queried locally without the network round-trip
        - Can be in-memory hash-table or an index on the local disk
        - Stream processor's local copy of the database needs to be kept up-to-date: With CDC, stream processor can subscribe to a changelog of the database as well as the stream of events. This effectively makes it a stream-stream join.
#### Table-table joins(Materialized view maintenance)
*Twitter timeline example:*
- When user wants to view their home timeline, it is too expensive to iterate over all the people th user is following, find their recent tweets, and merge them.
- Instead we want a timeline cache: a kind of "per-user" inbox to which twets are written as they are sent, so that reading the timeline is a single lookup.
- For materializing and maintaining this cache in a stream processor, we need streams of events for tweets and for follow relationships. the stream processor needs to maintain a database containing the set of ollowers for each user so that it knows which timelines need to be updated when a new tweet arrives.
- Another way of looking at this stream process is that it maintains a materialized view for a query that joins two tables(tweets and follows). The timelines are effectively the cache of the result of this query, updated every time the underlying tables change.
##### Time-dependence of joins
All the stream joins discussd above, maintain some state based on one joint, input and query that state on messages from the other join input.

The order of events that maintain this state is important.
- In a paritioned log, ordering is preserved within the partition but there is no ordering guarantee across partitions.
- If events on different streams happen around same time, in which order are they processed?
- If the ordering of events across streams is undetermined, the join becomes nondeterministic
- In data warehouses this issue is known as a *slowly changing dimension*(SCD), and it is often addressed by using a unique identifier for a particular version of the joined record and referencing that in the other side of the join. This makes the join deterministic, but has the disadvantage that log compaction is not possible, since all the versions of the records in the table need to be retained.

### Fault tolerance
The batch processing frameworks tolerate faults fairly easily by transparently retrying the failed tasks. It effectively provides exactly once processing. This is possible because:
- Input files are immutable
- Each task writes its output to a separate file on HDFS
- And output is made visible only when a task completes successfully.

Compared to batch processing, it is less straightforward to handle fault tolerance in stream processing: waiting until a task is finished before making it's output visible is not an option, because stream is infinite and we cannot finish processing it.
#### Microbatching and checkpointing
*Microbatching* breaks down the stream into small blocks, and treats each block as a miniature batch process.
- The approach is used in Spark Streaming
- The batch size is typically around 1 second
- Microbatching implicitly provides a tumbling window equal to the batch size(windowed by processing time).
- Any jobs that require larger windows need to explicitly carry over state from one microbatch to the next.

*Checkpointing* is an variant approach. We periodically generate rolling checkpoints of state and write them to durable storage.
- his approach is used in Apache Flink
- If a stream operator crashes, it can restart from its most recent checkpoint and discard any output generated between the last checkpoint and the crash.
- the checkpoints are triggered by the barriers in the message stream, similar to the boundaries in microbatches, but without forcing a particular window size.

Within the confines of stream processing framework, microbatching and checkpointing provide exactly once semantics. However, as soon as the output leaves the stream processor, the framework is no longer able to discard the output of the failed batch. In this case, restarting a failed task causes the external side effect to be applied twice.
#### Atomic commit revisited
In order to give appearance of exacly-once processingin te presence of faults, we need to ensure that all outputs and side effects of processing an event take place if and only if the processing is successful. Those effects include:
- Any messages sent to downstream operators or external messaging systems(inlcuding email or push notifications)
- Any database writes
- Any changes to operator state
- Any acknowledgement of input messages(including moving the consumer offset forward in a log-based message broker)

Those all things should happen atomically. 

Google Cloud Dataflow, VoltDB and Apached Kafka implement such stomic commit facility.
- Unlike XA, these implementations do not attempt to provide transactions across heterogenous technologies.
- They keep the transactions internal by managing state changes and messaging within the stream processing framework.
- the overhead of transaction processingcan be amortized by processing several input messages within a single transaction.
#### Idempotence
An idempotent operation can be performed multiple times with the same effect as if it was performed once.

Even if an operation is not naturally idempotent, it can be made idempotent with a bit of extra metadata. For example: 
- When consuming messages from Kafka, and writing a value to an external database, we can include the offset of the message that triggered the last write with the value.
- Thus we can tell whether an update has already been applied, and avoid performing the same update again.
- State handling in Storm's Trident follows similar idea.

Relying on idempotence implies several assumptions:
- Restarting a failed task must replay the same messages in the same order.
- The processing must be deterministic.
- No other code may concurrently update the same value.

Idempotent operations can be an effective way of achieving exactly once semantics with only a small overhead.
#### Rebuilding state after failure
Any stream process that requires state must ensure that this state can be recovered after failure.
- One option is to keep the state in a remote datastore and replicate it. But having to query remote database for each individual message can be slow.
 - An alternative is to keep state local to stream procesor, and replicate it periodically. Then when the stream processor is recovering from failure, the new task can read the replicated state and resume processing without data loss. For example:
    - Flink periodically captures snapshots of operator state and writes them to durble storage such as HDFS.
    - Samza and Kafka replicate state changes by sending them to a dedicated Kafka topic with log compaction, similar to CDC.
    - VoltDB replicates state by redundantly processing each message on several nodes.
- In some cases it may not even be necessary to replicate state, because it can be rebuilt from the input streams.

However, all of these trade-offs depend on the performance characteristics of the underlying infratructure.
- In some systems, network delay can be lower than disk access latency, and network bandwidth may be comparable to disk bandwidth.
- There is no universally ideal trade-off for all situations.
- Merits of local versus remote state may also shift as networking technologies evolve.



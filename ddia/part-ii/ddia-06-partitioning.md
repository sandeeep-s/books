# Partitioning
For very large datasets, or very high query throughput, replication is not sufficient: we need to break the data up into partitions(aka sharding).

**Terminological Confusion:** Partitioning is referred to by different names in different datastores.
- *Shard:* MongoDB, ElasticSearch, and SolrCloud
- *Region:* HBase
- *Tablet:* BigTable
- *vnode:* Cassandra, Riak
- *vBucket:* Couchbase

Partitions are defined in such a way that each piece of data(a record, row or a document) belongs to exactly one partition.

The main reason for partitioning data is *scalability*: 
- Different partitions can be placed on different nodes in a shared nothing cluster.Thus, a large dataset can be distributed across many disks, and the query load can be distributed across many processors.
- For queries that operate on a single partition, each node can independently execute the queries for its own partition, so query throughput can be scaled by adding more nodes.
- Large, complex queries can potentially be parallelized across many nodes, although this is significantly harder.

## Partitioning and Replication
Partitioning is usually combined with replication so that copies of each partition are stored on multiple nodes. A node may store more than one partition. Each partition's leader is assigned to one node and followers are assigned to other nodes. Each node may be a leader for some partitions and a follower for other partitions.

The choice of partitioning scheme is generally independent of replication scheme.

## Partitioning of Key-Value Data
*How do we decide, which records to store on which nodes?*

Our goal with partitioning is to spread the data and query load evenly across nodes.

**Skew:** 
- If the partitioning is unfair, so that some partitions have more data  or queries than others, we call it *skewed*. 
- The presence of skew makes partitioning much less effective.
- A partition with a highly disproportionate load is called a *hot spot*.
### Partitioning by Key Range
Assign a continuous range of keys(from some minimum to some maximum) to each partition.

- If we know the boundaries between the ranges, we can easily determine which partition contains a given key. 
- If you also know which partition is assigned to which node, we can make our request directly to the appropriate node.
- The ranges of data may not be evenly spaced, because your data may not be evenly distributed. In order to distribute the data evenly, the partition boundaries need to adapt to the data. 
- The partition boundaries might be chosen manually by an administrator, or the database can choose them automatically.

This partitioning strategy is used by
- Bigtable
- Its open source equivalent HBase
- Rethink DB
- MongoDB

Withing each partition we can keep the keys in sorted order: This has the advantage that range scans are easy and we can treat the key as a concatenated index in order to fetch several relaed records in one query.

The downside of key range partitioning is that certain access patterns can lead to hot spots.
### Partitioning by Hash of Key
Because of the risk of skew and hotspots, many distributed datastores use a hash function to determine the patition for a given key.

A good hash function takes skewed data and makes it uniformly distributed. For partitioning purposes, the hash function need not be cryptographically strong:
- MongoDB uses MD5
- Cassandra uses Murmur3
- Voldemort uses the Fowler-Noll-Vo function.

Once we have a suiotable hash function for keys, we can assign each partition a range of hashes, and every key whose hash falls within a partition's range will be stored in that partition.

The technique is good at distributing keys fairly among partitions. The partition boundaries can be evenly spaced, or they can be chosen pseudorandomly 

    Consistent Hashing: is a way of evenly distributing load across an internet-wide system of caches such as a CDN. It uses randomly chosen partition boundaries to avoid the need for central control or distributed consensus. This particukar approach doesn't work very well for databases, so it is rarely used in practice.

However, with hash partitioning the ability to do efficient range queries is lost. Keys that were once adjacent are now scatterred across all the partitions, so their sort order is lost.

- *MongoDB:* If we have enabled hash-based sharding mode, any range query has to be sent to all the partitions.
- *Riak, Couchbase, and Voldemart:*  Do not support range queries on the primary key.
- *Cassandra:*
    - Achieves compromise between the two partitioning strategies
    - A table in Cassandra can be declared with a *compound primary key* consisting of several columns
        - Only first part of that key is hashed to determine the partition
        - Other columns are used as a concatenated index for sorting data in Cassandra's SSTables.
        - A query therefore cannot search for a range of values within the first column of a compound key, but if it specifies a fixed value for the first column, it can perform an efficient range query over the other columns of the key.

The concatenated index approach enables an elegant data model for one-to-many relationships.
### Skewed Workloads and Relieving Hot Spots
Hashing a key to determine its partition can help reduce hpt spots. However, it can't avoid them entirely: in the extreme case where all reads and writes are for the same key, we still end up with all requests being routed to the same partition e.g. on a social media site, a celebrity user with millions of followersmay cause a storm of activity when they do something. This event can result in a large volume of writes to the same key(where key is perhaps teh ID of the celebrity or ID of the action people are commenting on). 

Today most data systems are not able to automatically compensate for such a highly skewed workload, so it's the responsibility of the application to reduce the skew.

## Partitioning and Secondary Indexes
A secondary index usually doesn't identify a record uniquely but rather is a way for searching occurences of a particular value.

- *Relational databases:* Secondary indexes are bread and butter of relational databases
- *Document databases:* Secondary indexes are common in document databases
- Key-value stores:
    - Many key-value stores(such as HBase and Voldemort) have avoided secondary indexes because of their added implementation complexity
    - But some (such as Riak) have started adding them because they are so usefiul for data modeling.

Problem with secondary indexes is that *they don't map neatly to partitions*.

There are two main approaches for partitioning a database with secondary indexes:
- Document based partitioning
- Term based partitioning
### Partitioning Secondary Indexes by Document
In this approach, each partition is completely separate: each partition maintains its own secondary indexes, covering only the documents in that partition. It doesn't care what data is stored in other partitions.
A document-partitioned index is also known as *local-index* because whenever we arite to the database, we only need to deal with the partition that contains the document id.

We need to use *scatter/gather* approach to querying a document partitioned database. We need to send the query to all the partitions, and combine all the results we get back. this can make read queries on secondary indexes quite expensive.Even if we query the partitions in parallel, scatter/gather is prone to tail-latency amplifications.

Document-partitioned secondary indexes are used by: MongoDB, Riak, Cassandra, ElasticSearch, SolrCloud and VoltDB.

Most DB vendors recommend that you structure you partitioning scheme so that secondary queries can be served from a single partition, but that is nopt always possible, especially when we are using multiple secondary indexes in a single query.
### Partitioning Secondary Indexes by Term
We can construct a *global index* that covers data in all partitions. This global index must also be partitioned, but it can partitioned differently than the primary index.

We call this a term-partitioned index, because the term we are loooking for determines the partition of the index. We can partition this index by the term itself or hash of the term.

Advantage of term-partitioned index over document partitioned index is that it can make reads more efficient: rather than doing a scatter/gather over all partitions, a client only needs to query the partition containing the term that it wants.

The downside of term-partitioned index is that teh writes are slower and more complicated: a write to a single document may now affectb multiple partitions of the index(every term in the document might be on a different partition, on a different node)

In practices, updates to global secondary indexes are often asynchronous.

Uses of global term-partitioned indexes
- Amazon DynamoDb
- Riak's search feature
- Oracle data warehouse: which ets you choose beteen local and global indexing scheme.

## Rebalancing Partitions
Over time things change in a database:
- The query throughput increases, so we want to add more CPUs to handle the load.
- The dataset size increases, so you want to add more disks and RAM to store it.
- A machine fails, and otehr machines need to take over the failed machine's responsibilities. 

All of these changes call for data and requests to be moved from one node to another. The process of moving load from one node in the cluster to another node is called *rebalancing*.

Rebalancing is usually expected to meet some minimum requirements:
- After rebalancing, the load(data storage, read and write requests) should be shared fairly between the nodes of the cluster.
- While rebalancing is happening, the database should continue to accept reads and writes.
- No more data than necessary should be moved between nodes, to make rebalancing fast and to minimize the network and disk I/O load.
### Strategies for Rebalancing
There are multiple ways of assigning partitions to nodes
#### How not to do it: hash mod N
Use *mod* operator.

*The problem with mod N approach:* If the number of nodes N changes, most of the keys will need to be moved from one node to another. Such frequent moves make rebalancing excessively expensive.
#### Fixed number of Partitions
Create many more partitions than there are nodes and assign several partitions to each node.

- Now, if a new node is added to the cluster, the new node can *steal* a few partitions from existing node until partitions are fairly distributed once again. If the node is removed from teh cluster, same happens in reverse.
- Only entire partitions are moved betwen nodes. The number of partitions does not change, and neither does the assignment of keys to partitions. The only thing that changes is the assign,ent of partitions to nodes.
- This change of assignment is not immediate--it takes some time to transfer a large amount of data over the network--so the old assignment of partitions is used for any reads and writes that happen while transfer is in progress.
- In principle, we can even account for mismatched hardware in our cluster: by assigning more partitions to the nodes that are more powerful, we can force those nodes to take a greater share of the load.
- This approach to rebalancing is used by: *Riak, ElasticSearch, Couchbase, Voldemort.*
- The number of partitions is usually fixed when the database is first set up and not changed afterward.
    - Although in principle, it is possible to split and merge partitions, *a fixed number of partitions is operationally simpler*, and so many fixed partitioned databases choose not to implement partition splitting.
    - The number of partitions configured at the outset is the maximum number of nodes you can have, so we need to choose it high enough to accomodate future growth.
    - However, each partition also has management overhead, so it's counterproductive to choose too high a number.
    - Choosing the right number of partitions is difficult if the total size of the dataset is highly variable.
        - Since each partition contains a fixed fraction of the total data, the size of each partition grows proportionally to the total amount of data in the cluster.
        - If partitions are very large rebalancing and recovery from node failures become expensive.
        - But if partitions are too small, they incur overhead
    - The best performance is achieved when the size of partitions is "just right", neither too big nor too small, which can be hard to achieve if the number of partitions is fixed but the dataset size varies.
#### Dynamic Partitioning
For key-range partitioned databases with fixed-partitioning, if we get the boundaried wrong, we could end up with all the data in one partition. Reconfiguring the partition boundaries manually would be very tedious.

For that reason key range partitioned databases such as HBase and RethinkDb create partitions dynamically.
- When a partition grows to exceed a configured size, it is split into two partitions so that approximately half of the data ends up on either side of the split.
- Coversely, if lots of data is deleted and a partition shrinks below some threshold, it can be merged with an adjacent partition.
- Each partition is assigned to one node, and each node can handle multiple partitions. After a large partition has been split, one of the two halves can be transferred to another node in order to balance the load.

Advantage of dynamic partitioning: The number of partitions adapts to the total data volume.

**Pre-Splitting:** A caveat is that an empty database starts off with a single partition, since there is no *a priori* information about where to draw the partition boundaries. While the dataset is small--until it hits the point at which the first partition is split--all writes have to be processed by a single node while other nodes sit idle.
- To mitigate this issue, HBase and MongoDB allow initial set of partitions to be configured on an empty database. This is called *pre-spliting*.
- In the case of key-range partitioning, pre-splitting requires that you already know what the key distribution is going to look like.
- Dynamic partitioning is suitable for both key-range and hash partitioned data.
- MongoDB supports both key-range and hash partitioning. In both cases it splits partitions dynamically.
#### Partitioning proportionally to nodes
*With Dynamic partitioning:* The number of partitions is proportional to the size of the dataset

*With Fixed Partitioning:* The size of each partition is proportional to the size of the dataset

In both the above cases, number of partitions is independent of the number of nodes.

A third option is to make the number of partitions proportional to the number of nodes.
- Have a fixed number of partitions per node
- The size of teh partitions grows proportionaly to dataset size while the number of nodes remains unchanged
- When we increase the number of nodes, the partitions become smaller again
- Since a larger data volume generally requires a larger number of nodes to store, this approach also keeps the size of each partition fairly stable.
- When a node joins teh cluster, it randomly chooses a fixed number of existing partitions to split, and then takes ownership of one half of each of those split partitions while leaving the otehr half of split partition in place.
- Picking partition boundaries randomly requires that hash-based partitioning is used.
- This approach is used by *Cassandra* and *Ketama*.
### Operations: Automatic or Manual Rebalancing
There is a gradient betwen fully automatic rebalancing and fully manual one. For example, Couchbase, Riak and Voldemort generate a suggested partitioning automatically, but require an administrator to commit it before it takes effect.

**Fully Automated Rebalancing**
- Can be convenient, because there is less operational work to do for normal maintenance,
- However, it can be unpredictable. Rebalancing is an expensive operation. If not done properly, it can overload network or the nodes and harm the performance of other requests while rebalancing is in progress.
- Such automation can be dangerous in combination with automatic failure detection.
- It can be a good thing to have human in the loop for rebalancing. It's slower than a fully automatic process, but it can help prevent operational surprises.

## Request Routing
With partitioned dataset, whena client wants to make a request how does it know which node to commect to?

**Service Discovery:**
- This is an instance of more general problem called *service discovery.*
- Any piece of software that is accesible over the network has this problem, especially if it is aiming for high availability(running in a redundant configuration on multiple machines).

On a high level, there are a few different approaches to this problem
*1. Allow clients to contact any node (e.g. via a round-robin load balancer):*
If that node coincidently owns the partition top which the request applies, it can handle the request directly; otherwise, it forwards the request to the appropriate node, receives the reply, and pases the reply along to the client.
*2. Send all requests from clients to a routing tier first:* 
The routing tier determines the node that should handle each request and forwards it accordingly. The routing tier itself does not nadle any requests; it only acts as a partition-aware load-balancer.
*3. Require that clients be aware of partitioning and the assignment of partitions to nodes.* 
In this case, the client can connect directly to the appropriate node without any intermediary.

In all the three cases the key problem is: how does the component making routing decisions learn about the changes to assignments of partitions to nodes.
- Many distributed data systems rely on a seperate coordination service such as ZooKeeper to keep track of this cluster metadata.Each node registers itself in ZooKeeper, and Zookeeper maintains an authoritative mapping of partitions to nodes. 
- Other actors such as the routing tier or the partitioning-aware client, can subscribe to this information in ZooKeeper.
- Whenever a partition changes ownership, or a node is added or removedZooKeeper notifies the routing tier so that it can keep it's routing information up to date.

**Practical Implementation:**
- *LinkedIn's Espresso:* Uses Helix for cluster management
- *HBase, SolrCloud and Kafka:* Use ZooKeeper to track partition assignment
- *MongoDB*: relies on it's own *config server* implementation and *mongos* daemons as the routing tier.
- Cassandra and Riak take a different approach:
    - They use *gossip protocol* among the nodes to disseminate any changes in the cluster state.
    - Requests can be sent to any node, and that node forwards them to the appropriate node for the requested partition.
    - This model puts more complexity in database nodes but avoids dependence on an external coordination service such as ZooKeeper.
- Couchbase: 
    - Does not rebalance automatically, which simplifies the design.
    - Normally it is configured with a routing tier called *moxi*, which learns about routing changes from the cluster nodes.

When using a routing tier or when sending requests to a random node, clients still need to find IP addresses to connect to. These are not as fast changing as assignment of partitions to nodes, so it is often sufficient to use DNS for this purpose.
### Parallel Query Execution
*Massively parallel processing*(MPP) relational database products, often used for analytics, are much more sohisticated(in comparison to most NoSQL distributed datastores) in the type of queries they support.

- A typical data warehouse query contains several join, filtering, grouping, and aggregation operations.
- The MPP query optimizer breaks this complex query into a number of execution stages and partitions, many of which can be executed in parallel on different nodes of a database cluster.
- Queries that involve scanning over large parts of a dataset particularly benefit from such parallel execution.








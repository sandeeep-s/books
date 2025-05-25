# Batch Processing
In *online systems* we care a lot about *response time*, so that a human user doesn't have to wait long for a response.

Let's distinguish between three different types of systems

**Services(*online systems*)**
- A service waits for request or instruction to arrive.
- Tries to handle received request or instruction as quickly as possible.
- *Response time* is usually the primary measure of performance of a service.
- *Availability* is very important

**Batch processing systems(*offline systems*)**
- A batch processing system takes a large amount of input data, runs a *job* to process it, and produces some output data.
- Jobs often take from a few minutes to days for completion, so there normally isn't a user waiting for job to finish.
- Batch jobs are often scheduled to run periodically.
- The primary performance measure of a batch job is *throughput*(the time it takes to crunch through an input dataset of certain size).

**Stream processing systems(*near real-time systems or nearline*)**
- Stream processing is somewhere between online and offline/batch processing systems.
- Like a batch processing system, a stream processor consumes inputs and produces outputs(rather than responding to requests).
- However, stream processor operates on events shortly after they happen, which allows stream processing systems to have lower latency than equivalemt batch processing systems.

*Working Set of a Job:* The amount of memory to which the job needs random access.

## Batch Processing with Unix Tools
A reminder about *unix philosophy* is worthwhile because the ideas and lessons from Unix carry over to large-scale, heterogenous data systems. 
### Simple Log Analysis
#### Chain of unix commands vs custom program
- Unix command pipeline relies on sorting a list of records in which multiple occurrences of the same record are simply repeated
- Custom program uses in-memory data structures  
#### Sorting versus in-memory aggregation
- If working set of the job is small enough, an in-memory data structure works fine.
- If the working set of the job is larger than the available memory, the sorting approach has the advantage that it can make efficient use of disks(like SSTables and LSM tress).
    - Chunks of data can be sorted and written out to disk on the segment files
    - Multiple sorted segments can be merged into a larger sorted file
    - Mergesort has sequential access patterns that perform well on disks(optimizng for sequential I/O theme from chapter 3)
    - Unix pipeline easily scales to large datasets, without running out of memory
### The Unix Philosophy
*Unix pipes: We should have some ways of connecting programs like a garden hose--screw in another segment when it becomes necessary to massage data in another way. This is the way of I/O also.*

The *unix phoilosophy* is the set of design principles popular among the developers and users of unix. It is described as follows:
1. Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new "features". 
2. Expect the output of every program to become the input to another, as yet unknown program. Don't clutter output with extraneous information. 
3. Avoid stringent columnar or binary input formats. 
4. Don't insist on interactive input.
5. Design and build software, even operating systems, to be tried early, ideally within weeks. Don't hesitate to throw away the clumsy parts and rebuild them.
6. Use tools in preference to unskilled help to lighten programming tasks, even if we have to detour to build the tools and expect to throw some of them away after you've finished using them.

This approach--automation, rapid prototyping, incremental iteration, being friendly to experimentation, and breaking down large projects into manageable chunks--sounds remarkably like Agile and DevOps movements of today.

A Unix shell like bash lets us easily compose small programs into surprisingly powerful data processing jobs. Even though many of the unix programs are written by different groups of people, they can be joined together in flexible ways. Unix enables this composability through: 
- A uniform interface
- Separation of logic and wiring
- Tranparency and experimentation
#### A uniform interface
If we want to be able to connect one program's output to another program's input, then all programs must use the same input/output interface. In Unix, that interface is a file(or more precisely, a file descriptor).
- A file is just an *ordered sequence of bytes*.
- Many different things can be represented using this same interface:
    - An actual file on the filesystem
    - A communication channel to another process(Unix socket, stdin, stdout)
    - A device driver(say /dev/audio or /dev/lp0)
    - A socket representing a TCP connection
    - And so on
- It's quite remarkable that these very different things can share a uniform interface, so they can easily be plugged together.
- By convention, many(but not all) Unix programs treat this sequence of bytes as ASCII text consisting of records seperated by the \n(newline) character. The fact that all these prigrams have standardized on using the same record seperator allows them to interoperate.
- The parsing of each record(i.e. a line of input is more vague). Unix tools commonly split lines into fields by whitespace or tab characters, commas, pipes, or other encodings.

*Balkanization of data:*
- Today, it's an exception, not the norm, to have programs that work together as smoothly as Unix tools do.
- Even the database with *same data model* do not make it easy to get data out of one and into the other.
- This lack of integration leads to Balkanization of data.
#### Seperation of logic and wiring
Another characteristic featire of Unix tools is their use of Standard Input(stdin) and Standard Output(stdout).
- By default stdin comes from keyboard and output goes to screen.
- We can also take input from a file, and/or redirect output to a file.
- Pipes let us attach stdout of one process to stdin of another process(with a small in-memory buffer and without writing the intermediate data stream to a disk).
- A program can still read and write files directly if it needs to, but Unix approach works best if a program doesn't worry about particular file paths and simply uses stdin and stdout.

One could say this is aform of loose coupling, late binding, or inversion of control.

Seperating the input/output wiring from the program logic makes it easier to compose small tools into bigger systems.

*Limits of stdin and stdout:*
- Programs that need multiple inputs or outputs are possible but tricky
- We can't pipe a program's output into a network connection
#### Transparency and experimentation
Part of what make Unix tools so successful is that they make it quite easy to see what is going on
- The input files to Unix commands are normally treated as immutable. This means you can run the commands as often as you want, trying various command-line options, without damaging the input files.
- We can end the pipeline at an point, pipe the output into less, and look at it to see if it has the expected form. This ability to inspect is great for debugging.
- We can write the output of one pipeline stage to a file and use that file as input to the next stage. This allows us to restart the later stage without rerunning the entire pipeline.

Unix tools remain amazingly useful, especially for experimentation. The biggest limitation of Unix tools is that they run on a single machine--and that's where tools like Haddop come in.

## MapReduce and Distributed Filesystems
MapReduce is a bit like Unix tools, but distributed across potentially thousands of machines. 
- Like Unix tools, it is a fairly blunt, brute-force, but surprisingly effective tool. 
- A single MapReduce jobis comparable to a single Unix process: it takes one or more inputs and produces one or more outputs.
- Running a MapReduce job normally does not modify the input and does not have any sideeffects other tham producing the output
- The output files are written once, in sequential fashion(not modifying any existing part of a file once it is written)
- While Unix tools use stdin and stdout as input and output, MapReduce jobs read and write files on a distributed filesystem.
- Examples of distributed file systems
    - HDFS,GlusterFS, QFS
    - Object storage services like Amazon S3, Azure Blob Storage, and OpenStack Swift

*HDFS:*
- In Hadoop's implementation of MapReduce, the distributed filesystem is HDFS.
- HDFS is an open source reimplementation of Google File System(GFS).
- HDFS is based on *shared nothing principle*, in contrast to teh shared-dosk approach of *NAS* and *SAN* architectures.
- HDFS consists of a daemon process running on each machine, exposing a network service allowing other nodes to access files stored on that machine.
- A central server called *NameNode* keeps track of which file blocks are stored on which machine.
- Thus, HDFS conceptually creates one big filesystem that can use the space on the disk of al machines running the daemon. 
- In order to tolerate machine and disk failures, file blocks are replicated on multiple machines.
- File access and replication are done over a conventional datacenter network without special hardware(in contrast to NAS, SAN or RAID which use special hardware)
### MapReduce Job Execution
MapReduce is a programming framework with which we can write code to process large datasets in a distributed filesystem like HDFS.

*Pattern of data processing in MapReduce:*
1. Read a set of input files, and break it up ino *records*.
2. Call the *mapper function* to extract a *key and value* from each input record.
3. Sort all the key-value pairs by key.
4. Call th reducer function to iterate over the sorted key-value pairs.(If there are multiple occurrences of the same key, the sorting has made them adjacent in the list, so it is easy to combine those values without keeping a lot of state in memory.

Those four steps can be performed by one MapReduce job. Steps 2(map) and 4(reduce) are where we write our custom data processing code. Step 1 is handled by the input format parser. Step 3(sort) is implicit in MapReduce--the output from mapper is always sorted before it is given to the reducer.

To create a Mapreduce job we need to implement two callback functions:
1. *Mapper*
    - The mapper is called once for every input record.
    - Its job is to extract the key and value from the input record.
    - For each input, it may generate any number of key-value pairs(including none).
    - It does not keep any state from one input record to the next, soeach record is handled independently.
2. *Reducer:*
    - The MapReduce framework takes the key-value pairs produced by mappers.
    - It collects all the values belonging to the same key.
    - Then it calls the reducer with an iterator over that collection of values.
    - The reducer can produce output records.

Viewed like this:
- The role of the mapper is to prepare the data by putting it intoa form that is suitable for sorting
- The role of the reducer is to process data that has been sorted
### Distributed execution of MapReduce
MapReduce can parallelize a computation across many machines, without having to write code to explicitly handle parallelism.
- The mapper and reducer operate on only one record at a time.
- They don't need to know where their input is coming from or their output is going to, so the framework can handle the complexities of moving data between machines.

In Hadoop MapReduce parallelization is based on partitioning
- Map tasks:
    - The input to a job is typically a directory in HDFS
    - Each file or file block within the directory is considered to be a seperate partition.
    - Each partition can be processed by a separate map task.
    - The MapReduce scheduler tries to run each mapper on one of the machines that stores replica of th input file, provided that the machine has enough spare RAM and CPU resources to run the map task. This principle is known as *putting the computation near the data*: it saves copying the input file over the network, reducing network load and increasing locality.
    - The reduce side of the computation is also partitioned. 
    - To ensure all key-value pairs with same key end up at the same reducer, the framework uses a hash of the key to determine which reduce task should receive a particular key-value pair.
- Shuffle: The process of partitioning by reducers, sorting and copying data [artitions from mappers to reducers is called shuffle. 
    - Due to the likely large size of the dataset, sorting is performed in stages.
    - First, each map task partitions its output by reducer, based on the hash of the key.
    - Each of these partitions is written to a sorted file on the mapper's local disk.
    - Once mapper finishes reading its input file and writing sorted output files, the MapReduce scheduler notifies the reducers.
    - The reducers connect to each of the mappers and download the files of sorted key-value pairs for their partition.
- Reduce tasks:
    - The reduce task takes the files from the mappers and merges them together, preserving the sort order.
    - The reducer is called with a key and iterator that scans over all the records with the same key.
    - The reducer can use arbitrary logic to process these records.
    - The reducer can generate any number of output records.
    - These output records are written to a file on the distributed filesystem.
### MapReduce Workflows
It is very common for MapReduce jobs to be joined together in workflows, such that output from one job becomes the input for another job.

Hadoop MapReduce framework does not have explicit support for workflows. The chaining can be doine implicitly by directory name: 
- The first job cn be configured to write its output to a designated directory in HDFS and the second job reads its input from the same directory.
- This is more like a sequence of commands, where each command's output is written to a temporary file, and the next command reads from the temporary file.

*Workflow schedulers:*
- One job in a MapReduce workflow can only start when prior jobs have completed.
- To handle these dependencies between job executions, various workflow schedulers for Hadoop have been developed: including Oozie, Azkaban, Luigi, Airflow, and Pinball.
- These schedulers also have management features that are useful when maintaining a large collection of batch jobs.

Other high-level tools for Hadoop: Pig, Hive, Cascading, Crunch, and FlumeJava. Tool support is important for managing complex dataflows.
### Reduce-Side Joins and Grouping
In many datasets, one record msy be associated with another record through a reference such as:
- *Foreign key* in a releational database
- *Document reference* in a document database
- *Edge* in a graph database

A join is necessary when some code needs access to the records on both sides of that association.

When we talk about joins in case of batch processing, we mean resolving all occurrences of some association within the dataset. 
For example, a join between a log of user activity events and a database of user profiles.
- Take a copy of user database and put it in the same distributed filesystem as log of user activity events.
- We would then have user database in one set of files in HDFS and the user activity records in another set of files.
- We could use MapReduce to bring all the relevant records in the same place and process them efficiently.
#### Sort-merge Joins
In sort-merge join algorithm, mapper output is sorted by key and reducers then merge together the sorted list of records from both sides of the join.
##### Bringing related data together in one place
- In a sort-merge join, the mappers and the sorting process make sure that all the necessary data for performing the join operation for a particular entity id is brought together in the same place: a single call to the reducer. 
- Having lined up all the required data in advance, the reducer can be a fairly simple, single-threaded piece of code that can churn through records with high throughput and low memory overhead.

*Messaging:*
One way of looking at this architecture is that mappers send messages to the reducers.
- The key acts as the destination address to which the value should be delivered.
- All key-value pairs with the same key will be delivered to the same destination.

*Seperated the physical network communication from application logic:*
- Using the MapReduce programming model has seperated the physical network communication aspects of the computation(getting the data to the right machine) from the application logic(processing the data once we have it). 
- This separation contrasts with the typical use of databases, where a request to fetch the data from a database occurs somewhere deep inside the application code.
- Since MapReduce handles all network communication, it also shields the application code from having to worry about partial failures, such as crash of another node: MapReduce transparently retries failed tasks without affecting the application logic.
#### Group By
Another common use of "bringing the related data in the same place" pattern is grouping records by some key.
- All the records with the same key form a group
- The next step is to perform some kind of aggregation within the group.

*Implementation in MapReduce:*
- Set up the mappers so that the key-value pairs they produce use the desired grouping key.
- The partitioning and sorting process then brings together all the records with same key in the same reducer.

**Common use of group by: *Sessionization:***
Sessionization is the process of collating all the activity events for a particular user session, in order to find out the sequence of actions the user took.
- If we have multiple web servers handling user requests, the activity events for a particular user are most likely scattered across various different server's log files.
- We can implement sessionization by using a session cookie, user ID, or similar identifier as the grouping key and bringing all the activity events for a particular user together in one place.
- Different user's events are distributed across different partitions
#### Handling Skew
*Linchpin Objects or Hot Keys:*
- The pattern of "bringing all records with the same key to the same place breaks down" if there is very large amount of data related to the same key.
- For example, few celebrities with millions of followers on social network
- Such dosproportionately active database records are known as linchpin objects or hot keys.
- A single reducer may become a hotspot--that is one reducer must process significantly more records than the others.
- Since MapReduce job is complete only when all of its mappers and reducers have completed, any sibsequent jobs must wait for the slowest reducer to complete before they can start.
##### Compensating Algorithmsfor Hot Spots:
- *The skewed-join method in Pig:*
    - First runs a sampling job to determine which keys are hot.
    - When performing the actual join, the mappers send records related to a hot key to one of several reducers, chosen at random.
    - For the other input to the join, the records relating to that hot key need to be replicated to all the reducers handling that key.
    - The technique spreads the work of handling hot keys to several reducers, which allows it to be parallelized better, at the cost of having to replicate the other join inout to multiple reducers.
- *The sharded join method in Crunch:*
    - It is similar to the skewd joined method, but requires hot keys to be specified explicity instead of using a sampling job.
- *Hive's skewed join method:*
    - It requires hot keys to be specified explicitly  in the table metadata.
    - It stores records related to those keys in separate files from the rest.
    - It uses *map-side join* for hot keys.

When grouping grouping and aggregating records by hot key, we can perform grouping in two stages.
- The first MapReduce stage sends records to a random reducer.
- Each reducer performs grouping on a subset of records for the hot key and outouts a more compact aggregated value per key.
- The second MapReduce job then combines the values from all of the first-stage reducers into a single value per key.
### Map Side Joins
*Reduce-side joins pros and cons:*
- The reduce-side joins have the adavantage that we do not need to make any assumptions about the input data:whatever its properties and structure, mappers can prepare the data to be ready for joining.
- The downside of reduce-side joins is that all the sorting, copying to reducer inputs and merging of reducer inputs can be quite expensive.

If we can make certain assumptions about the input data, it is possible to make joins faster by using *map-side joins.*
- This approach uses a cut-down MapReduce job  without reducers and sorting.
- Instead each mapper simply reads one input file block from the distributed filesystem and writes one output file to the filesystem.
#### Broadcast Hash Joins
- Use when a large dataset is joined with a small dataset.
- Small dataset must be able to be loaded entirely into memory in each of the mappers.

How does it work?
- When a mapper starts up, it can first read the smaller database from the distributed filesystem into an in-memory hash table.
- The mapper can then scan over the larger dataset and simply lookup by key in the hashtable. 
- There can still be several map tasks: one for each file block of the large input to the join.

The word *broadcast* reflects the fact that the small input is "broadcast" to all partitions of the larger input.

An alternative is store the small input in a read-only index on the local disk. The frequently used parts of this inde will remain in OS cache, so this approach can provide random access lookups almost as fast as an in-memory hastable, but without actually requiring the dataset to fit into the memory.
#### Partitioned merge joins
- The inputs to the map-side join are partitioned in the same way: with the records assigned to partitions based on the same hash key and the same hash function.
- Same numbered partitions are joined using hash join, with smaller sized partition of the two loaded into in-memory hashtable.
- All the records to join will be located in the same numbered partiton. Then each mapper can load just a small amount of data in the hashtable.
- Known as *bucketed map joins* in Hive
#### Map-side merge joins
- Datasets are partitioned in the same way and *sorted* based on the same key.
- It does not mtter if datasets fit in memeory, because the mapper performs the same merging operation that would be performed by a reducer: reading both input files sequentially, in order of ascending keys, and matching records with the same key.
#### MapReduce workflows with map-side joins
The choice of map-side or reduce-side join affects the structure of the output.
- The output of a reduced-side join is partitioned and sorted by the join key.
- The output of map-side join is partitioned and sorted in the same way as large input.

Map-side joins also make aassumptions about the size, sorting and partitioning of input datasets.
- Knowing about the physical layout of datasets in the distributed filesystem becomes important when optimizing join strategies:
    - It is not sufficient to know the encoding format and the directory name
    -  We must also know the number of partitions and the keys by which the data is partitioned and sorted
- In Hadoop ecosystem, sych metadata is stored in HCatalog or Hive metastore.

### The Output of Batch Workflows
Batch processing is neither transaction processing or analytics. 
- It is closer to analytics. in that a batch process typically scans over large portions of an input dataset.
- The output of a batch process in not a report but some other kind of structure.
#### Building search indexes
If we need to perform a full-text search on a fixed set of documents, then a batch process is a very effective way of building indexes:
- The mappers partition the set of documents as needed.
- Each reducer builds the index for its partition.
- The index files are written to the distributed filesystem.
- Building such document partitioned index parallizes very well.
- Since querying the search index by keyword is a read-only operation, these index files are immutable once created.
- If the indexed set of documents changes there are two options
    1. Rerun the entire indexing workflow for entire set of documents. The approach can be computationally expensive but easy to reason about: documents in, indexes out.
    2. Build indexes incrementally: if we want to add, remove or update documents in an index, Lucene writes out new segment files and asynchronously merges and compacts segment files in the background.
#### Key value stores as batch process output
Other common uses for batch processing are to build
-  Machine learning systems e.g. spam filters, anomaly detection, image recognition
- Recommendation systems e.g. people we may know, products we may be interested in, or related searches

The output of those batch jobs is some kind of a database. These databases need to be queried from the web application that handles user requests, which is usually seperate from teh Hadoop infrastructure.

##### Getting output of a batch process into a database
*Write to the database using the client library directly from mapper or reducer, one record at a time*
    
This approach has several drawbacks:
- Making network request for every single record in order of magnitudes slower than the normal throughput of a batch task. Even if the client library supports batching, performance is likely to be poor.
- The database can be easily overwhelmed if mappers or reducers concurrently write to the same output database. Consequently, performance for queries can suffer and cause operational problems in other parts of the system.
- MapReduce provides clean all-or-nothing guarantee for the job output. But writing to an external system from inside the job produces externally visible side effects which cannot be rolled back in case of job failures. Thus we have to worry about the results from partially completed jobs being visible to other systems, and the complexities of Hadoop task attempts and speculative execution.

*Building a brand new database inside the batch job* as files in distributed filesystem
- A much better solution is to build a brand new database *inside* the batch job and write it as files to the distributed filesystem.
- Those data files are then immutable once written, and can be loaded in bulk into servers that handle read-only queries.
- Various key-value stores support building database files in MapReduce jobs: including Voldemort, Terrapin, ElephantDB, and HBase bulk loading.
- Building these database files is a good use of MapReduce:
    - Using a a mapper to extract a key and then sorting by that key is a lot of the work required to build an index.
    - Since most of the key-value stores are read-only, the data structures are quite simple.
- The server can continues serving requests to the old data files, while the new files are copied to the server's local disk. Onc the copying is complete, the server atomically switches over to querying the new files. If anything goes wrong, it can switch back to old files again.
#### Philosophy of batch process outputs
The handling of output from MapReduce jobs follows UNIX philosophy: it encourages experimentation by being very explicit about dataflow.
- A program reads its input and writes its output.
- The input is left unchanged.
- Any previous output is completely replaced with the new output.
- There are no side effects.

By treating onputs as immutable and avoiding side effects(such as writing to external databases), batch jobs both achieve good peformance and become much easier to maintain.
- *Human fault tolerance:* If the output is wrong or corrupted due to buggy code, we can simply roll back to the previous version of code and rerun the job, or keep the old output in a different directory and just switch back to it. Rolling back the code in satabases with read-write transactions will do nothing to fix the data in the database.
- *Minimizing irreversibility:* As a consequence of ease of this rolling back, feature development can proceed more quickly than in an environment in which mistakes could mean irreversible damage.
- The map or reduce task failures due to transient issues ca be tolerated and retries. The automatic retry is only safe because inputs are immutable and outputs from failed tasks are discarded.
- The same set of files can be used as input to different jobs e.g. monitoring jobs that calculate and evalauate if output has expected characteristics.
- MapReduce jobs seperate logic from wiring(configuring the input and output directories). This provides separation of concerns and possible reuse of code: one team can focus on implementing a job that does one thing well, while other teams can decide where and when to run the job.

*Difference between UNIX tools and Hadoop:*
- Because UNIX tools assume untyped text files, they have to do a lot of input parsing.
- Hadoop eliminates thes low value syntactic conversions by using more structured file formats: e.g. Avro and Parquet provide efficient schema based encoding and allow schema evolution over time.

### Comparing Hadoop to Distributed Databases
*Massively parallel processing*(MPP) databases focus on parallel execution of analytic SQL queries on a cluster of machines.

Combination of MapRdeuce and a distributed filesystem provides something much more like a general-purpose operating system that can run arbitrary programs.
#### Diversity of storage
Databases require us to structure data according to a particular model(e.g. relational or document) e.g. MMP databases typically require careful upfront modeling of data and query patterns before importing the data into databases porprietery format.

On the other hand files in a distributed filesystems are just byte sequences that can be written using any data model and encoding. Thus, Hadoop opened the possibility of indiscriminately dumping the data into HDFS. 
- In practice, making the data available quickly is often more useful than trying to decide on the ideal data model upfront. 
- Collecting data in its raw form and worrying about schema design later, allows the data collection to be speeded up(a concept sometimes known as *data lake* or *enterprise data hub*).
- Indiscriminate data dumping shifts the burden of interpreting the data
    - Instead of forcing the producer to of the dataset to bring it into standerdized format, the interpretation of data becomes consumers problem(the *schema-on-read* approach).
    - This can be an advantage if producers and consumers are separtae teams with different priorities.
    - There may not be even one ideal data model, but different views onto the data that are suitable for different purposes.
    - *The Sushi principle*:"raw data is better"--simply dumping data in its raw form sloows for different transformations.

Thus Hadoop has often been used for imp[lementing ETL processes:
- Data from transaction processing systems is dumped into the distributed filesystem in some raw form.
- Then MapReduce jobs are written to clean up that data, transform them into relational form and import it into an MPP data warehouse for analytic purposes.
- Data modeling still happens, but it is ina seperate step , decoupled from data collection.
- This decoupling is possible because a distributed filesystem supports data encoded in any format.
#### Diversity of processing models
MPP databases are monolithic, tightly integrated pieces of software:
- They can take of storage layout on disk, query planning, scheduling and execution.
- SQL query language allows expressive queries and elegant semantics without writing code, making it accessible for graphical processing tools used by BAs(such as Tableau)

However, not all kinds of processing can be sensibly expressed as SQL queries. 
- For example:
    - Machine learning and recommendatio systems
    - Full-text seach indexes with relevance ranking models
    - Performing image Analysis
- Such processing requires a more general model of data processing.
- These kinds of processing are often very specific to a particular application, so they inevitable require writng code, not just queries.

MapReduce gave engineers the ability to easily run their own code over large datasets. 
- Due to the opennes of te Hadoop platform, it was feasible to implement a whole range of processing models on top of MapReduce(including SQL model).
- Those various processing models can all be run on a single shared-use cluster of machines, all accessing the same files on the distributed filesystem.
    - In Hadoop system, there is no need to import data into several specialized systems for different kinds of processing: the system is flexible enough to support diverse set of workloads within the same cluster.
    - Not having to move around data makes it a lot easier to dervive value from data, and a lot easier to experiment with new processing models.
    - For example, HBase(OLTP database) and Impala(MPP-style analytic database), both use HDFS for storage.They are very different approaches to accessing and processing data, but they can nevertheless co-exist and can be integrated into the same system.
#### Designing for frequent faults
MapReduce and MPP databases differ in the design approach towards *handling of faults*, and *the use of memory and disk*.

*MPP Databases:*
- If node crashes while query is executing, MPP databases abort the entire query, and either let user resubmit it, or automatically rerun it.
- MPP database also prefer to keep as much data as posiible in memory

*MapReduce:*
- MapReduce can tolerate failure of a map or reduce task without it affecting the job as whole, by retrying work at the granularity of a task.
- It is also very eager to write data disk, partky for fault tolerance and partly assuming the large size of the dataset.
- The MapReduce approach is appropriate for larger jobs: jobs that process so much data and run for such a long time that they are likely to experience at least one task failure along the way.
- MapReduce is designed to tolerate frequent unexpected termination(e.g. in a computing cluster where the freedom to arbitrarily terminate low priority processes enables better resource utilization).

In an environment where tasks are not so often terminated, the design decisions of MapReduce make less sense.

## Beyond MapReduce
MapReduce is just one among many different processing models for distributed systems. Depending on the volume of data, the stucture of the data and the type of processing being done on it, other tools may be moe appropriate for expressing a computation.

Implementing complex processing jobs using raw MapReduce APIs is quite hard and laborious.
- Various higher-level programming models(Pig, Hive, Cascading, Crunch) were created as abstractions on top of Mapreduce.
- Their higher level constructs make implementing many common batch processing tasks significantly easier.

However, there are also problems with the MapReduce excution model itself.
- These manifest themselves as poor performance for some kinds of processing.
- These cannot be fixed by adding another level of abstraction.

MapReduce is very robust, but on the other hand, other tools are sometimes orders of magnitudes faster for some kinds of processing.
### Materialization of Intermediate State.
In MapReduce, the main contact points of a job with rest of the world are its input and output directories on the distributed filesystem. 
- This setup is reasonable if we want to reuse output from one job as input to several different jobs(including jobs developed by other teams)
- Publishing data to a well-known location in the distributed filesystem allows loose coupling, so that jobs don't know who is producing their input and consuming their output.
- *Materialization:* 
    - In many cases, the output of one job is used at input to one other job, which is maintained by the same team. In this case, the files on the distributed filesystem are simply *intermediate state*: means of apssing data from one job to the next.
    - The process of writing this intermediate state to files is called *materialization*(It means to eagerly compute the result of some operation and write it out, rather than computing it on demand when requested).

Mapreduce's approach of fully materializing the intermediate state has downsides compared to Unix pipes:
- MapReduce job has to wait until all the preceding jobs have completed, thus slowing down the execution of the workflow as a whole. Skew and varying load on different machines can result in a few straggler tasks. Processes connected by a Unix pipe are started at the same time, with output being consumed as soon as it is produced.
- Mappers are often redundant. In many cases, the mapper code could be part of the previous reducer, and reducers could be chained directly.
- Replication of files, resulting from storing intermidiate data on a distributed filesystem is an overkill, doe to the temporary nature of this data.
#### Dataflow engines
Dataflow engines such as Spark, Tex, and Flink were developed to resolve these problems with MapReduce. 
- They handle entire workflow as one job, rather than breaking it up into multiple subjobs.
- They explicitly model the flow of data through several processing stages.
- Like MapReduce:
    - They work by repeatedly calling a user-defined function to process one record at a time, on a single thread.
    - They parallelize work by partitioning inputs.
    - They copy the output of one function over the network to become the input to another function.
- Unlike MapReduce:
    - These functions need not take strict roles of alternating map and reduce.
    - They can be assembled in more flexible ways.
    - We call these functions operators.

Dataflow engines provide different options for connecting one operators output to another's in put.
- Repartition and sort records by key.
- Take several inputs and paritition them in the same way, but skip sorting. This saves effort on partitioned hash joins.
- For broadcast hash joins, the same output from one operator can be sent to all partitions of the join operator

This style of processing engine is  based on research systems like *Dryad* and *Nephele*. It offers several advantages over MapReduce:
- Expensive work such as sorting need only be performed in places where it is actually required.
- There are no unnecessary map tasks, since work done by a mapper can be inorporated into the preceding reduce operator(because the mapper does not change partitioning of the dataset).
- Because all joins and data dependencies in a workflow are explicitly declared, the scheduler has an overview of what data is required where, so it can make locality optimizations.
- Dataflow generalizes the idea of storing all intermediate state to be kept in memoryor written to local disk, which requires less I/O than writing to HDFS.
- Operators can start executing as soon as their input is ready.
- Exisitng JVM proceses can be reused to run new operators, reducing startup overheads of MapReduce(which launches a new JVM for each task)

Since operators are a generalization of map and reduce, the same processing code from MapReduce workflows can run on a dataflow engine: workflows implemented in Pig, Hive or Cascading can be switched from MapReduce to Tez or Spark with a simple configuration change, without modifying code.

Tez is a fairly thin library that depends on YARN shuffle service for the actual copying of data between nodes. Spark and Flink are big frameworks that include their own communication layer, scheduler, and user facing APIs. 
#### Fault tolerance
In MapReduce, if a task fails,it can just be restarted on another machine and read the same input again from the filesystem.

Dataflow tools take a different approach to fault tolerance: if  machine fails and the intermediate state on that machine is lost, it is recomputed from the other data that is still available.
- To enable this recomputation, the framework must keep track of how a given pience of data was computed--which input partitions were used and which operators were applied to it.
    - Spark uses the resilient doistributed dataset(RDD) abstraction for tracking the ancestry of data
    - Flink checkpoints operator state.
- When recomputing data, it is important to know if the computation is deterministic. It is better to make operators deterministic, in order to avoid cascading faults.

Recovering from faults by recomputing is not always the right answer: if the intermediate data is much smaller than the source data, or of the computation is very CPU-intensive, it is probably cheaper to materialize the intermediate data to files than to recompute it.
#### Discussion of Materialization
MapReduce is like writing the output of each command to a temporary file, whereas dataflow engines look more like Unix pipes. 

When using a dataflow engine, materialized datasets on HDFS are still usually the inputs and final outputs of a job. Like with MapReduce the inputs are immutable and the output is completely replaced. The improvement over MapReduce is that we save ourself writing all the intermediate state to the filesystem as well.
### Graphs and Iterative Processing
For graphs, the goal in a batch processing contexts is to perform some kind of offline processing or analysis on the entire graph. For example, PageRank is one of the most famous graph analysis algorithms.

Many graph algorithms are expressed by traversing one edge at a time, joining one vertex with an adjacent vertex in order to propagate some information, and repeating until some condition is met--e.g. there are no more edges to follow or some metric converges. This kind of algorithm is called a *transitive closure*.

Implementing such an algorithm in a MapReduce style is very inefficient.
- It is possible to store a graph in a distributed filesystem(in files containing lists of vertices and edges).
- But the idea of "repeating until done" cannot be expressed in plain MapReduce, since it performs only a single pass over the data.
- It can be implementing in MapReduce using iterative style by running one batch process in each iteration. But this is inefficient because MapReduce does not account for the iterative nature of the algorithm: it will always read the entire input dataset and produce a completely new output dataset, even if only a small part of the graph has changed compared to the last iteration.
#### The Pregel Processing Model
As an optimization for batch processing graphs, the *bulk synchronous parallel*(BSP) has become popular. It is implemented by Apache Giraph, Spark's GraphX API, and Flink's Gelly API. 

Idea behind Pregel is:
- One vertex can send message to another vertex, and typically those messages are sent along the edges in a graph.
- In each iteration, a function is called for each vertex, passing the function all the messages that were sent to that vertex--much like a call to the reducer.
- The difference from MapReduce is that in the Pregel model, a vertex remembers it's state in memory from one iteration to the next, so the function only needs to process new incoming messages.
- If no messages are being sent in some part of the graph, no work needs to be done.

It's bit similar to the actor model if we think of each vertex as an actor, except thatvertex state and messages between vertices are fault tolerant and durable, and communication proceeeds in fixed rounds: at every iteration,the framework delivers all the messages sent in the previous iteration.
#### Fault Tolerance
The fact that vertices can only communicate via message passing(not by querying each other directly) helps improve the performance of Pregel jobs, since messages can be batched and there is less waiting tiome for communication.

Pregel implementations gurantee exact once message processing at destination vertices. The framework transparently recovers from faults in order to simplify the programming model for algorithms on top of Pregel. This fault tolerance is achoieved by periodically checkpointing the state of vertices at the end of an iteration--i.e. writing their full state to a durable storage.
#### Parallel Execution
*Thinking like a Vertex:*
- A vertex does not need to know on which physical machine it is executing. When it sends messages to other vertices, it sends them only to a vertex ID. 
- It is upto the framework to partition the graph--i.e. to decide which vertex runs on which machine, and how to route messages over the network so that they end up in the right place.
- Because the programming model deals with just one vertex at a time(sometimes called "thinking ike a vertex"), the framework may partition the graph in arbitrary ways.

The overhead of sending messages over the network may significantly slow down distributed graph algorithms.
- Graph algorithms often have a lot of cross-machine communication overhead
- The intermediate state(messages sent between nodes) is often bigger than the original graph.
- If our graph can fit in memory on a single computer, it is highly likely that a single machine(maybe even single-threaded) algorithm will outperform a distributed batch process. Even if the graph is bigger than memory, it can fit on the disks of a single computer, single-machine processing using a framework such as GraphChi is a viable option. 

### High-level APIs and Languages
 Problem of physically operating distributed batch processes at large scale(many petabytes of data on clusters of over 1000 machines) has been more or less solved. Attention has now turned to other areas:
 - Improving the programming model
 - Improving the efficiency of processing
 - Broadening the set of problems that these technologies can solved

Examples of high level languages and APIs
 - High level languages and APIs such as Pig, Hive, Cascading, and Crunch became popular because programming MapReduce jobs by hand is laborious.
 - Tez allowed seamlesly moving the jobs written in these high level languages to a new dataflow execution engine.
 - Spark and Flink have their own high-level dataflow engines inspired from FlumeJava.

These dataflow APIs generally use relational-style building blocks to express a computation.:
- Joining datasets on the value of some fields
- Grouping tuples by key
- Filtering by some condition
- Aggregating tuples by counting, summing or other functions.
 Internally, these oprations are implemented using the various join and grouping algorithms already discussed.

*Advantages of high-level interfaces:*
- Require less code
- Allow interactive use
    - We write analysis code incrementally in a shell and run it frequently to observe what it is doing. 
    - This style of development is very helpful when exploring a dataset and experimenting with approaches for processing it.
- Make humans using the system more productive
- Improve job execution efficiency at the machine level
#### The move towards declarative query languages
Specifying joins as relational operators in a *declarative way* rather than code allows automatic query optimization. Hive, Spark, and Flink have cost-based query optimizers.

However, unlike the fully declarative query model of SQL, the MapReduce and its dataflow successors can execute arbitrary code in the callback functions to decide the output. This allows us to draw upon a large ecosystem of existing libraries to do the parsing, natural language analysis, image analysis, and running numerical or statistical algorithms.

Dataflow engines have incorporated more declarative features beyond joins while retaining their fexibility advantage of running arbitrary code.
#### Specialization for different domains
Batch processing engines and MPPs are being used for distributed execution of algorithms from an increasingly wide range of domains. This results in many common cases where standard processing patterns keep on reoccuring, and it is worth to have reusable implementations of the common building blcoks. For example:
- Business intelligence analysis and business reporting
- Statistical and numerical algorithms, which are needed for machine learning algorithms such as classification and recommendation systems.
    - Mahout implements various algorithms for machine learning on top of MapReduce, Spark, and Flink.
    - MADlib implements similar functionality inside a relational MPP database(Apache HAWQ).
- Spacial algorithms such as *k-nearest neighbours*
- Genome analysis algorithms







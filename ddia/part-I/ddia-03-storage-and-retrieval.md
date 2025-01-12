# Storage and Retrieval
On the most fundanental level, database must do two things
1. Store the data, when you give it some data
2. Give the data back to you, when you ask it again later

An application developers we should understand how database internally stores and retrives data in order To be able to:
- Select a storage engine that is appropriate for our application
- Tune the storage engine to perform well on our kind of workload

For example there is a big difference in the storage engines optimized for transactional workloads and those that are optimized for analytics.

## Data Structures that Power our Database
**Log:** 
- Log is an append only sequence of records.
- *Appending to a file is generally very efficient.* 
- Many databases internally use a *log*, which is an append-only data file. 
- Though the basic orinciple is the same, Real databases have more issues to deal with such as: 	- Concurrency control
	- Reclaiming disk space so that log doesn't grow forever
	- Handling errors and partially written records

**Index:**
- An *index* helps to efficeintly find the value for a particular key in the database. 
- Indexes keep some additional metadata on the side, that acts as a signpost and helps us locate the data we want.
- An index is an additional structure that is derived from the primary data. 
- Any kind of index usually slows down writes as index also needs to be updated every time the data is written.
	- Well-chosen indexes speed up queries but slow down writes. 
	- For this reason, database don't usually index everythoing by default.
	- Application developer or database administrator is requiredi to choose indexs manually, using their knowledge of the application's typical query patterns.
### Hash Indexes
- Data storage consists of only appending to a file. 
- We keep an in-memory hashmap where every key is mapped to a byte offset in the data file--the location at which the value can be found. 
- Whenever you append a new key-value pair to the file, you also update the hashmap to reflect the offset of the data you just wrote(this applies both to inserts and updates). 
- When you want to look up a value, use the hashmap to find the offset in the data file, seek to that location and read the value.

Storage engines using hash indexes can offer high-performance reads and writes subject to the requirement that all keys fit into available RAM. Values can be loaded from disk with just one disk seek. If that part of the datafile is already in filesystem cache, no disk I/O is required at all.

Such storage engines are well suited to situations where there are large number of writes(updates) per key, but there are not too many disticnt keys.
#### Recaliming Disk Space
*Segments*: 
- Break up a log file into segemnts by closing a segment file when it reaches a certain size, and make subsequent writes to a new segment file. 
- Each segment has its own in-memory hashmap. 
- In order to find the value for a key, we check first in the most recent segment's hashmap; if the key is not present we check the second one and so on.

*Compaction*: Throwing away duplicate keys in the log and keeping only the most recent update for each key

*Merge*: Combining several compacted segments together into a new segment file

Merging and compaction of segments can be done in background thread, while we continue toe serve read requests using old segment files, and write requests to the latest segment file. After merging process is complete we can switch read requests to use new segment files.
#### Hash Index Implementation Considerations
*File format:* Use a binary format that first encodes the length of a string in bytes, followed by the raw string.

*Deleting records:* Append a *tombstone*(A special deletion record) to the data file, which tells the merging process to discard any previous values for the key.

*Crash recovery:* Storing a snapshot of each segment's hashmap on disk, so that they can be loaded into memory quickly in case database restarts.

*Partially written records:* Use checksums to detect corrupted parts of the log to be detected and ignored.

*Concurency control:*  
- Have only one writer thread as writes are appended to the log file in strictly sequential order. 
-  They can be read concurrently by multiple threads, as data-file segments are append-only and otherwise immutable.
#### Benefits of append-only log design 
- Appending and merging are sequential write operations which are much fater than radom writes, especially on magnetic spinning-disk hard drives. Sequential writes are also preferrable on SSDs. 
- Concurrency and crash recovery are much simpler if segment files are append-only or immutable.
- Merging old segments avoids the problem of data files getting fragmented over time.
#### Drawbacks of Hash table indexes
- It won't work well for a very large no. of keys.
- range queries are not efficient
### SSTables and LSM-Trees
#### SSTable(Sorted String Table)
In SSTable format, records in segment files are sorted by key. With this format, we cannot append new recordsto the segment immediately, since writes can occur in any order.
##### Benefits of SSTables over Hash Index
- Merging segments is simple and efficient, using *mergesort* like algorithm.
- In-memory index can be sparse, containing offsets of only a few keys: one key for every few kilobytes and use scanning to find key after locating a proximate key.
- Since read-requests need to scan over several key-value pairs in the requested range anyway, it is possible to group those records in a block and compress it before writing to the disk.
#### LSM Tree (Log Structured Merge Tree)
Maintaining a sorted structure in memory is much easier(red-black trees or AVL trees) than maintaining it on disk(B-Trees). The storage engine can work as follows:
- Add write to an in-memory balanced tree data structure(aka *memtable*).
- When the memtable gets bigger than a specified threshold--typically a few megabytes--write it out to disk as an SSTable.
    - This can be done efficiently because the tree already maintains the key-value pairs sorted by key. 
    - The new SSTable file becomes the most recent segment ofthe database.
    - While the SSTable is being written out to disk, writes can continue to anew memtable instance
- In order to serve a read request, first try to find the key in memtable, then in the most recent on-disk segment, then in the next older segment and so on
- From time to time run a merging and compaction process in the background.
- To avoid losing writes which are in the memtable but not yet written to disk(e.g. in case of a database crash), we can keep a seperate log on disk to which every write is immediately appended. Everytime a memtable is written to disk, the corresponding log file can be deleted.

Storage engines that are based on the principle of merging and compacting sorted files are often called *LSM storage engines*.
##### Performance Optimizations
- *Bloom Filters:* LSM trees can be slow when looking up keys that do not exist in database. Bloom filters are a memory-efficient data structure for approximating the contents of a set. It can tell us if a key does not appear in the database, and thus save many unnecessary disk reads for non-existent keys.
- *Compaction and Merging Strategies:*
    - *Size-tiered compaction:* Newer and smaller SSTables are successively merged into older and larger SSTables.
    - *Leveled compaction:* The key range is split up into smaller SSTables and older data is moved into seperate "levels", which allows the compaction to proceed more incrementally and use less disk space.

The basic idea of LSM trees--keeping a cascade of SSTables that are merged in the background--is simple and effective. 
- Even when the dataset is much bigger than the available memory it continues to work well.
- Since data is stored in sorted order you can efficiently perform range queries
- Because the disk writes are sequential the LSM-Tree can support remarkably high write throughput.
### B-Trees
B-Trees are the most widely used indexing structure. It is a standard index implementation in all relational databases, and many non-relational databases use them too.

- B-Trees keep key-value pairs sorted by key, which allows efficient key-value lookup and range queries. 
- B-Trees break the database down into fixed size blocks or pages, traditionally 4 kb in size, and read or write one page at a time. Each page can be identified using an address or a location, which allows one page to refer to another.
- One page is designated as the root of the B-Tree; whenever we look up a key in the index, we start here.
- The page contains several keys and references to child pages.
- Each child is responsible for a continuous range of keys, and the keys between the references indicate where the boundaries between those ranges lie.
- As we walk down the tree, we eventually reach a leaf page containing individual keys., which either contains the value for each key inline or contains references to the pages where the values can be found.

*Branching factor:* The number of references to child pages in one page of a B-Tree.  It depends on the amount of space required to store page references and the range boundaries. Typically it is several hundred.

#### Update and Insert Operations
*Updating a value for existing key in B-Tree:* Search for the leaf page containing that key, change the value in that page, and write the page back to disk. All references to the page remain valid.

*Inserting a new key in B-Tree:*
- Find a page whose range encompasses the new key, change the value in that page, and write the page back to disk.
- If there isn't enough free space in the page to accomodate the new key, it is split into two half full pages, and the parent page is updated to account for the new subdivision of key ranges. 

This algorithm ensures the tree remains balanced: a B-Tree with n keys always has a depth of O(log n). Most databases can fit into a tree that is 3-4 levels deep.
#### Making B-Trees reliable
- Include a *write-ahead log* (aks redo log): this is an append-only file to which every B-Tree modification must be written before it can be applied to the pages of the tree itself.
- Protect the tree's data structures with latches(lightweight locks
#### B-Tree Optimizations
- Use *copy-on-write* scheme: A modified page is written to a different location and a new version of the parent pages in the tree is created, pointing at the new location. This is also useful for concurrency control.
- Lay out the tree so that leaf pages appear in sequential locations on the disk
- Additional pointers have been added to the tree. e.g. Sibling pages to the left and right, which allows scanning keys in order without jumping back to parent pages 
- Fractal trees
### Comparing B-Trees and LSM Trees
*Writes and Read:* LSM-Trees are typically faster for writes, while B-Trees are thought to be faster for reads. Reads are typically slower on LSM-Trees because they have to check several different data structures and SSTables at different stages of compaction.
#### Advantages of LSM Trees
*Write Throughput:* 

LSM trees are typically able to sustain higher write throughput than B-Trees because:
- They sometimes have lower write amplification (*Write Amplification:* One write to the database resulting in multiple writes to the disk over the course of databases lifetime. In write-heavy applications, write amplification has a direct cost: the more that storage engine writes to the disk, the fewer writes it can handle within the avilable disk bandwidth.)
- They sequentially write compact SSTable files rather than having to overwrite several pages in the tree. 

*Compression:*
- LSM-tres can be compressed better, and thus often produce smaller files on disk than B-Trees.

*Magnetic Hard Drives:*
- On Magnetic hard drives sequential writes are much faster than random writes.

*SSDs:*
- On SSDs, the impact of the storage engine's write pattern is less pronounced as firmware internally uses log-structured alogirthms to convert random writes into sequential writes
- However lower write amplification and reduced fragmentation are still advantageous on SSDs: representing data more compactly allows more read and write requests within the available I/O bandwidth.
#### Downsides of LSM Trees
Compaction process can sometimes interfere in the performance of ongoing reads and writes. 
- Disk's finite write bandwidth needs to be shared between initial write(logging and flushing memtable to disk) and the compaction threads running in the background. 
	- Since disks have limited resources, a request may have to wait while the disk finishes an expensive compaction operation.	
	- When writing to an empty database, the full disk bandwidth can be used for the initial write but the bigger the database gets, the more disk bandwidth is required for compaction.
- At high percentiles, the response time of queries to log-structured storage engines can be quite high, and B-Trees can be more predictable
- If write throughput is high, compaction may not be able to keep up with the rate of incoming writes. 
	- In this case, no. of unmerged segments keeps on growing until you run out of disk space.
	- Reads also slow down beacuse more segments have to be checked
#### Advantages of B-Trees
- In a B-Tree index, each key exists in exactly on place in the index, whereas a log-structured storage engine may have multiple copies of the same key in different segments. 
	- This aspect makes B-Trees attractive in databases that want to offer strong transactional semantics. 
	- In a B-Tree index, transaction isolation can be implemented by attaching locks directly to the tree.

There is no quick or easy rule for determining which type of storage engine is best for your use case, so it is worth testing empirically.
### Other Indexing Structures
*Secondary Indexes:* Both B-Trees and log-structured indexes can be used as secondary indexes.
#### Storing values within the Index
*Heap file:*
The value in the index can be either the actual row(document, vertex) or a reference to the row stored in a *heap file*. The heap file approach is prefered as it avoids duplicating data when multiple secondary indexes are present.

*Clustered Index:* Stores the indexed row directly within index.

*Covering Index (aka Index with Included Columns):* Stores some of table's columns within the index

Clustered and covering indexes can speed up reads but they require additional storage and can add overhead to writes.
#### Multi-Column Indexes
Needed while querying multiple columns of a table

*Concatenated Index:* Combines several fields into one key by appending one column to another. Order of field concatenation is decided by index definition.

*Multi-dimensional indexes:* More general way of querying several columns at once. Particulary useful for geospatial data but not limited to that. Are commonly implemented by using specialized spatial indexes such as R-Trees.
#### Full-text Search and Fuzy Indexes
Full-text search engines commonly allowa search for one word to be expanded 
- To include synonyms of the word
- To ignore grammatical variations of the word
- To search for occurences of words near each other in the same document 
- Support various other features that depend on liguistic analysis of the text

To cope with typos in documents or queries, full-text search engines allow searching for text with an edit-distance.

Other fuzzy search techniques go in the direction of *document classification* and *machine learning*.
#### Keeping evrything in memory
Compared to main memory disks are awkward to deal with. With both magnetic disks and SSDS, data needs to be laid out carefully if we want good peformance reads and writes. 

But disks also have significant advantages:
- They are durable
- They have lower cost per GB than RAM

**In-Memory Databases:**
 As RAM becomes cheaper, cost per GB argument is eroded. Many datasets are not that big making it feasible to keep them entirely within memory, potentially distributed across several machines.

 *Caching only in-memory stores:* Used where it is acceptable for data to be lost if the machine is restarted.
 - Products
    - MemCached

 *Durable in-memory stores:* Durability for in-memory databases can be achieved :
 - With special hardware(such as battery-powered RAM)
 - By writing a log of changes to disk
 - By writing periodic snapshots to disk
 - By replicating in-memory state to other machines
 - Products:
    - *VoltDB, MemSQL, Oracle Times Ten*: In-memory databases with relational model.
    - *RamCloud:* In-mempry key-value store with durability.
    - *Redis and Couchbase:* Provide weak durability by writing to disk asynchronously.

*In-memory databases can be faster, not because they do not need to read from the disk, but because they can avoid encoding in-memory data structures in a form that can be written to disk.*

Besides performance, another interesting area for in-memory databases is *providing data models that are difficult to implement with disk-based indexes*.

*Anti-caching approach:* 
- Can be used to extend in-memory databases to support datasets that are larger than available memory.  
- It works by evicting the least recently used data from memory to disk when there is not enough memory, and loading it back into memory when it is read again in future.
- This is similar to what operating systems do with virtual memory and swap files, but the database can manage memory more efficiently than OS, as it can work at granularity of individual records than entire memory pages.

*Non-volatile Memory Technologies:* Might affect storage engine design in future

## Transaction Processing or Analytics?
**Transaction Processing**
- Means allowing clients to make low latency reads and writes
- Basic access pattern:
    - An application typically looks up a small number of records by some key, using an index.
    - Records are inserted or updated based on the user's input
    - It is also called onlie transaction processing(OLTP)
- Properties of OLTP systems
    - *Main read pattern:* Small no. of records per query, fetched by key
    - *Main write pattern:* Random-access low-latency writes from user input
    - *Primarily used by:* End user/Customer via a web application
    - *What data represents:* Latest state of data(current point in time)
    - *Dataset Size:* Gigabytes to Terabytes

**Data Analytics:**
- Usually an analytic query needs to scan a huge number of records, only reading a few columns per record, and calculates aggregate statistics(such as count,sum or average) rather than returning raw data to the user.
- These queries are usually written by business analysts and feed into reports that help the management of company make better decisions(*business intelligence*)
- Also called as online analytics processing(OLAP)
- Properties of OLAP systems
    - *Main read pattern:* Aggregate over large number of records.
    - *Main write pattern:* Bulk import(ETL), or Event Stream
    - *Primarily used by:* Internal analyst, for decision support
    - *What data represents:* History of events that happened over time
    - *Dataset Size:* Terabytes to Petabytes
### Data Warehousing
A data warehouse is a separate database that contains the read only copy of the data in all the various OLTP systems in the company.

Advantages of using data warehouse separate than OLTP systems:
- It can be optimized for analytic access patterns.
- Analysts can query it without affecting OLTP operations.

*Extract-Transform-Load(ETL)* is used to get the data from OLTP systems into the data ware house. Data is extracted from OLTP databases(either by using periodic data dumps or a continuous stream of updates), transformed into an analysis-friendly schema, cleaned up and then loaded into the data warehouse.

#### The divergence between OLTP databases and data warehouses
The data model of data warehouse is most commonly relational because SQL is generally  a good fit for analytic queries. 

*Even though data warehouse and relational OLTP both have an SQL interface, the internals of the systems can look quite different, because they are optimized for very different access patterns.*

*Batch Processing* means running jobs periodically. 
### Schemas for Analytics: Stars and Snowflakes
#### Star Schema(aka Dimensional Modeling)
In analytics there is much less diversity of data models. Many data warehouses are used in a fairly formulaic style known as *star schema*(aka dimensional modeling)
- At the centre of the schema is a so-called *fact table*. 
    - Each row of the fact table represents an event that happened at a particuar time. 
    - Capturing facts as individual events allows maximum flexibility of analysis later but can also make event table extremely large.
- Some columns in the fact table are attributes(or values). Other columns in facyt tables are refrences to other tables called as *dimension tables*.
- The *dimensions* represent the who, what, where, when, how, and why of the event.
#### Snowflake Schema
Snowflake schema is a variation of star schema , where dimensions are further broken down into subdimensions.

Snowflake schemas are more noirmalized than star schemas, but star schemas are often preferred because they are simpler for analysts to work with.

### Column Oriented Storage
In a typical data warehouse, columns are often very wide (100 of columns).

If there are trillions of rows and petabytes of data in your database, storing and querying them efficiently becomes a challenging problem.

In most OLTP databases, storage is laid out in a row-oriented fashion. In Document databases as well, an entire document is typically storaed as one continguous sequence of bytes.

The idea behind column-oriented storage is: don't store all the values from one row together, but store all the values from each column together instead. If each column is stored in a seperate file, a query only needs to read and parse only those columns that are used in that query, which can save a lot of work.

The column oriented storage relies on each column file containing the rows in same order.
#### Column Compression
**Bitmap Encoding:** Often, the number of distinct values in columns is small compared to the number of rows. We can turn a column with *n* distinct values and turn it into n seperate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is one if row has that value, and 0 if not. If *n* is bigger, the bitmaps can additionally be *run-length encoded* to make the encoding of the column remarkably compact.
#### Memory Bandwidth
For data warehouse queries that need to scna millions of rows, a big bottleneck is the bandwidth for getting data from disk into memory. 

Developers of analytical database also worry about:
- Efficiently using the bandwidth from main memory into the CPU cache
- Avoiding branch mispredictions and bubbles in the CPU instruction pieline
- Making use of single-instruction-multi-data(SIMD) instructions in the modern CPUs
#### Vectorized Processing
Besides reducing the volume of data that needs to be loaded from a disk, column-oriented storage layouts are also good for making efficient use of CPU cycles. 
- Column compression allows more rows of a column to fit in the same amount of L1 cache.
- Operators such as bitwise AND and OR can be designed to operate on such chunks of compresed column data directly. 
- This technique is known as *vectorized processing*.
#### Sort Order in Column Storage
Sorting in columns store can be used as an indexing mechanism.

Sorted order can help with compression of columns. If the primary sort column does not have many distinct values, it will have long sequences where the same value is repeated many times in a row. A simple run length encodingcould compress that column to a few kilobytes--even if the table has billions of rows. The compression effect is strongest on the first sort key.Columns further down the sorting priority won't compress that well.
##### Several different sort orders
Having multiple sort orders in a column-oriented store is a bit similar to having multiple secondary indexes in a row oriented store. Big difference is secondary indexes are pointers to the data in row-oriented store, while column-oriented stores they are only columns with values.
#### Writing to Column Oriented Storage
Column-oriented storage, compression, and sorting all help to make read queries easier for analytics use cases. However they make writes more difficult. An update-in-place approach, like B-Trees is not possible with compressed columns.

*LSM Trees* are a good solution for write efficiency.
- All writes first go to an in-memory store, where they are added to a sorted structure and prepared for writing to disk.
- When enough writes have accumulated, they are merged with column files on disk and written to new files in bulk.
- Queries need to examine both the column data on disk and the recent writes in memory, and combine the two. However, query optimizer hides this distinction from the user.
### Aggregation: Data Cubes and Materialized Views
Data warehouse queries often involve an aggregate function, such as COUNT, SUM, AVG, MIN or MAX in SQL. Caching results of most often used aggregate queries can save us some wasteful crunching of raw data.

#### Materialized Views
Materialized Views can be used to cache the results of the aggregate queries. They are defined as standard(virtual views). But while *vitual view* is just a shortcut for writing queries, materialized view is an actual copy of the query results, written to the disk.
##### Data Cubes
Data cube(aka OLAP cube) are a special case of a materialized view. It is a grid of aggregates grouped by different dimensions.

The advantage of materialized data cubes is that certain queries become very fast because they have effectively been precomputed.

The disadvantage is that a data cube doesn't have the same flexibility as querying the raw data. Most data warehouses therefore  try to keep as much raw data as possible, and use aggregates such as data cubes only as performance boost for certain queries.


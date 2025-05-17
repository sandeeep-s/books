# Data Models and Query Languages

## Data Models
*Data models are the most important part of developing software* because they have such a profound effect on:
- How the software is written and 
- How we think about the problem we are solving. 

**Most applications are built by layering one data model on top of the other.** 
- For each layer, the key question is, how is it represented in the terms of its next lower layer. 
- Each layer hides the complexity of lower layers by providing a clean data model. 
- These abstractions allow different groups of people--for example the engineers at database vendor and application developers using the database--to work together effectively.

Every data model embodies assumptions about how it is going to be used. 
- Some kinds of usages are easy and some are not supported
- Some operations are fast and some perform badly
- Some data transformations are natural and some are awkward. 

*Since the **data model has such a profound effect on what the software above it can and cannot do**, it is important to choose one that is appropriate to the application.*
### Relational Model Vs Document Model
The roots of relational databases lie in: 
- Business data processing: typically, *transaction processing*(entering sales or banking transactions, airline reservations, stock-keeping in warehouses)
- And *batch processing*(customer invoicing, payroll, reporting)
#### NoSQL Databases
Driving forces behind the adoption of NoSQL databases
- **A need for greater scalability** than relational databases can achieve, including *very large datasets* and *very high write throughput*
- A widespread **preference for free and open source products** over commercial database products
- **Specialized query operations** that are not well-supported by relational models
- Frustration with **restrictiveness of relational schema**, and a desire for a **more dynamic and flexible data model**

**Polyglot Persistence:** refers to using relational databases alongside a broad set of nonrelational datastores 
#### The Object-Relational Mismatch
Also called *impedance mismatch*, it is the disconnect between object-oriented and relational models requiring an awkward translation layer between the two.

ORM frameworks reduce the amount of bilerplate code required for this translation layer, but they can't completely hide differences between the two models.

*JSON Representations:*
- For data structure like a resume, which is a self contained document, a JSON representation might be quite appropriate. 
- JSON representation seemingly reduces impedance mismatch betweeen application code and storage layer. 
- The JSON representation has a better locality than a relational multi-table schema. 
#### One-to-Many Relationships
The one-to-many relationships imply a tree structure in the data. Document representation(JSON or XML) makes this tree structure explicit. 

One-to-many relationships can be represented in various ways in SQL:
- Multiple tables with foreign key references
- Structured datatypes, JSON and XML data which allow multi-valued data to be stored within a single row, with support for querying and indexing inside those documents
- Encode data as JSON or XML document, and store it in a text column on the database, and let the application interpret its structure and content.
#### Many-to-One and Many-to-Many Relationships
###### Normalization
Whether we store a ID(reference) or text string for standardized data is a question of duplication. 
- When you store the text directly, you are duplicating the human-meaningful information in every record that uses it. 
- When we use an ID, the information that is meaningful to humans is stored in only one place, and everything that refers to it uses an ID(which only has meaning within the database).

Anything that is meaningful to humans may need to change sometime in future-- and if the information is duplicated, all the redundant copies need to be updated.That incurs write overheads, and risks inconsistencies. Removing such duplication is the key idea behind normalization in databases. The advantage of using an ID is that because it has no meaning to humans, it never needs to change: the ID can remain the same even if the information it represents changes

Normalizing data requires *many-to-one* relationships, which:
- Are natural in relational databases  because joins are easy. 
- But do not fit nicely in document databases as support for joins is usually weak. If the database itself doesn't support joins, you have to emulate a join in application code by making multiple queries to the database.
#### Relational verus Document Databases today
Main arguments in favor of each model are
- Document data model:
    - Schema flexibility
    - Better performance due to locality
    - That for some applications, it is closer to the data structures used by the application.
- Relational data model:
    - Better support for joins
    - Better support for many-to-one and many-to-many relationships
###### Schema
*Schema-on-read:*
- The schema is implicit and only interpreted by clients when data is read.
- Document databases are *schema-on-read*:
- Schema-on-read is advantageous if the data is heterogenous(items in the collection don't have the same structure). Few exaample this is the case when:
    - There are many different types of objects, and it is not practicable to put each type of object in its own table.
    - The structure of the data is determined by external systems over which you have no control and which may change any time.

*Schema-on-write:*
- The schema is explicit and the database ensures all written data conforms to it.
- Traditional approach of relational databases is *schema-on-write*:
- Cases where all records are expected to have the same structure, schemas are a useful mechanism for documenting and enforcing that structure.

This is similar to dynamic vs strong typing in programming languages.

###### Storage Locality
A document is usually stored as a single contiguous string. If the application often needs to access the entire document, there is a performance advantage to the storage locality.

The locality advantage only applies if we need large parts of the document at the same time. 
- On updates to a document, the entire document usually needs to be rewritten. 
- For these reasons it is usually recommended to keep documents fairly small and avoid writes that increase the size of a document.

If data is split across multiple tables, multiple index loopkups are required to retrive it all, which may require more disk seeks and take more time.
###### Impedance mismatch with application data structures
It seems relational and document databases are becoming similar over time as the data models complement each other. If the database is able to handle document-like data and also perform relational queries on it, applications can use the combination of features that best fits their needs.
###### Highly interconnected data
The document model is awkward, the relational model is acceptable, and graph models are the most natural.

## Query Languages for Data Models
### Declarative Query Languages
SQL is a *declarative language*. It fairly closely follows the structure of *relational algebra* for querying data

In a *declarative language* (like SQL or relational algebra), you just specify the pattern of the data you want--what conditions the result must meet, and how you want the data to be transformed(e.g. sorted, grouped, and aggregated)--but not **how** to achieve that goal.

A declarative query language is attractive because
- Tt is typically more **concise and easier to work with** than an imperative API. 
- More importantly, it also **hides implementation details** of database engine, which makes it possible for a database system to introduce performance improvementswithout requiring any changes to queries.
- Fact that SQL is limited in functionality gives the database much more room for **automatic optimizations**.
- Declarative languages often lend themselves to **parallel execution**, because they specify only the patterns of the results, not the algorithm that is used to determine the results(which might imply a particular order of the operations making oit hard to parallelize).

### MapReduce Querying
Mapreduce is a **programming model for processing large amounts of data in bulk across many machines**.

MapReduce is neither a declarative query langauge nor a fully imperative API, but somewhere in betwen: 
- Logic of the query is expressed with snippets of code, which are called repeatedly by the processing framework. 
- It is based on map and reduce functions available in many programming languages.

The *map* and *reduce* functions are somewhat restricted in what they are allowed to do. 
- They must be pure functions, which means they only use the data that is passed to them as input, they cannot perform additional database queries, and they must not have any side effects.
- These restrictions allow the database to run the functions anywhere, in any order and rerun them on failure.

MapReduce is a fairly low-level programming model for distributed execution on cluster of machines. A usability problem with MapReduce is that you have to write two carefully coordinated Javascript functions, which is often harder than writing a single query.

## Graph-Like Data Models
As connections within your data become more complex, it becomes more natural to start modeling your data as a graph.

A graph consists of two kinds of objects: **vertices**(_nodes or entities_) and **edges**(_relationships or arcs_).Well known algorithms can operate on these graphs.
Typical examples of data that can be modelled as a graph
- Social graphs
- The web graph
- Road and rail networks

Graphs can both **store homogeneous and heterogeneous data**.
### Structuring and querying data in graphs
#### Property Graph Model
- Implemented by **Neo4j**, **Titan**, and **InfiniteGraph**
- Each vertex consists of
    - A unique identifier
    - A set of outgoing edges
    - A set of incoming edges
    - A collection of properties(key-value pairs)
- Each edge consists of
    - A unique identifier
    - The vertex at which the edge starts(tail vertex)
    - The vertex at which the edge ends(head vertex)
    - A label to describe the kind of relationship betwen the two vertices
    - A collection of properties(key-value pairs)
- Any vertex can have an edge connecting it with any other vertex. There is **no schema which restricts which kind of things can or cannot be associated**.
- Given any vertex, you can efficiently find both it's incoming and outgoing edges, and thus **traverse the graph**
- By using different **labels for different kind of relationships**, you can store many different kind of information in a single graph, while still maintaining a clean data model.
- Graphs are **good for evolvability**: as you add features to your application a graph can easily be extended to accommodate changes in your application's structures.
- Property Graph Model is **queried by using a declarative query language**
#### Triple Store Model
- Implemented by **Datomic**, **AllegroGraph** and others
- Triple-store model is mostly **equivalent to the property graph model**, using different words to describe the same ideas. 
- In a triple-store, all information is stored in the form of very simple three-part statements -- **subject, predicate, object**. 
- The subject of a triple is equivalent to a vertex in the graph
- Object is one of two things: 
    1. *A value in a primitive data type:*
        In this case the, the predicate and object are equivalent to the key and value property on the subject vertex
    2. *Another vertex in the graph:*
        In that case, the predicate is an edge in the graph, the subject is the tail vertex, and the object is the head vertex. 






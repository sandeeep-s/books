# Part III. Derived Data
In large applications we often need to be able to access and process data in many different ways. There is no one database that can satisfy those different needs simultaneously. Applications thus commonly use a combination of several different datastores, indexes, caches, analytics streams, etc. and implement mechanisms for moving data from one store to another.

**Integrating multiple different data systems, potentially with different data models and optimized for different access patterns, into one coherent application architecture is one of the most important things that needs to be done in a nontrivial application.**

## Systems of Record and Derived Data
On a high level, systems that store and process data can be grouped into two broad categories
1. *Systems of record* (aka *Source of truth*)
    - A system of record holds the authoritative version of our data.
    - When new data comes in, it is first written here
    - Each fact is represented exactly once(the representatio is typically normalized)
    - If there is a dicrepancy between system of record and another system, then the value in system of record is the correct one.
2. *Derived data systems*
    - Data in a derived system is the result of taking some existing data from another system and transforming or processing it in some way.
    - If we lose derived data, we can recreate it from the original source.
    - Derived data is *redundant*, in the sense that it duplicates existing information. However it is often essential for getting *good performance on read queries.*
    - It is *denormalized*.
    - Denormalized values, caches, indexes, materialized views all fall in this category
    - In recommendation systems, predictive summary data is often derived from usage logs.
    - We can **derive different datasets from a single source**, enabling us to **look at the data from different *points of view***.

Making a clear distinction between systems of record and derived data clarifies the dataflow through our system: it makes explicit which parts of the system have which inputs and which outputs, and how they depend on each other.

Most databases, storage engines, and query languages are not inherently either a system of record or a dervived system. They are just tools. The distinction between the system of record and derived data system depends on how we use the tools in our application.
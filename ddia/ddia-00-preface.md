# Designing Data Intensive Applications

## Preface
Recent buzzwords relating to storage and processing of data
- NoSQL
- Big Data
- Web-scale
- Sharding
- Eventual Consistency
- ACID
- CAP Theorem
- Cloud Services
- MapReduce
- Real-time

Recently we have seen interesting developments in databases, distributed systems and the way applications are built on top of them. Driving forces behind these developments are:
- Internet companies are handling large amounts of data and traffic, forcing them to create new tools that enable them to efficiently handle such scale
- Businesses need to
    - Be agile
    - Test hyptheses cheaply
    - Respond quickly to market insights by keeping:
        - Development cycles short and
        - Data models flexible
- Free and open-source software has become very successfull and is now preferred to commercial or bespoke in-house software in many environments
- CPU clocks are barely getting faster, but multi-core processors are standard, and networks are getting faster. This means parallelism is only going to increase
- Even if you are on a small team, you can now build systems that are distributed across many machines and even multiple geographic regions, thanks to IaaS.
- Many services are now expected to be highly available; extended downtimes due to outages or maintenance is becoming increasingly unacceptable.

We call an application **data-intensive** if the data is its primary challenge--the quantity of data, complexity of data, or the speed at which the data is changing--as opposed to **compute-intensive** where CPU cycles are the bottleneck

Important tools and technologies for data storeage and processing 
- RDBMS
- NoSQL
- Message Queues
- Caches
- Search indexes
- Frameworks for batch and stream processing

As software engineers and architects, we need to have a technically accurate and precise understanding of the various technologies and their trade-offs if we want to build good applications.

Behind the rapid changes in technology, there are enduring principles that remain true. Understanding those principles helps us see where each tool fits in, how to make good use of it, and how to avoid it's pitfalls.

### Goals of the book
- Help you navigate the diverse and fast changing landcape of technologies for processing and storing data.
- Find useful ways of thinking about data systems--not just how they work, but also why they work that way, and what questions we need to ask.
- Enable us to decide what kind of technology is appropriate for which purpose
- Understand how tools can be combined to form the foundation of a good application architecture
- Develop a good intuition for what your systems are doing under the hood so that we can reason about their behavior, make good design decisions, and track down any problems that may arise.
- Learn how to make data systems scalable, for example to support web or mobile apps with millions of users
- Learn how to make applications hihgly available(minimize downtime) and operationally robust
- Learn ways of making systems easier to maintain in the long run, even as they grow and as requirements and technologies change.
- Know what goes on inside majpor websites and online services
# Distributed Data

There are various reasons we might want to **distribute a database across multiple machines**
- **Scalability:** If our data volume, write load, or read load grows bigger than a single machine can handle, we can potentially **spread the load** across multiple machines.
- **Fault Tolerance/Availability:** If our applications need to continue working even if one machine (or several machines, or the network, or an entire datacenter) goes down, we can use multiple machines for **redundancy**. If one goes down, another one can take over.
- **Latency:** If you have users around the world,you might want to have servers at various locations worldwide so that each user can be served from a datacenter that is **geographically close** to them. That avoids the users having to wait for network packets to travel halfway around the world.

## Scaling to Higher Load
### Shared Memory Architecture (Vertical Scaling or Scaling Up)
- Many CPUs, many RAM chips, and many disks can be joined together under one operating system
- A fast interconnect allows any CPU to access any part of the memory or disk
- All the components can be treated as a single machine

*Problems with Shared Memory Architecture:*
- Cost grows faster than linearly
- Due to bottlenecks, a machine twice the size cannot necessarily handle twice the load
- It may offer limited fault tolerance (through hot-swappable components).
- It is limited to a single geographic location
### Shared Disk Architecture
- Uses several machines with independent CPUs and RAM, but stores data on an array of disks that is shared between the machines, which are connected via a fast network.

*Problems with Shared Memory Architecture:*
- Contention
- Overhead of locking
### Shared Nothing Architecture(Horizontal Scaling or Scaling Out)
- Each machine or virtual machine running the database software is called a node
- Each node uses it's CPUs, RAM, and disks independently
- Any coordination between the nodes is done at the software level, using a conventional network

*Advantages of Shared Nothing Architecture:*
- No special hardware is required by a shared-nothing system, so we can use whatever machines have the best price/performance ratio.
- We can potentially distribute data across multiple geographic regions, and thus reduce latency for users and potentially be able to survive the loss of a datacenter.
- With cloud deployments of virtual machines, even for small companies, multi-region distributed architecture is now feasible.
- They are not necessarily the best choice for every use case, but require a lot of caution from application developers.
- They usually also incur additional cpmplexity and limit the expressiveness of the data models we can use.

## Replication vs Partitioning
There are two common ways data is distributed across multiple nodes
### 1. Replication
- Keeping the copy of same data on multiple nodes, potentially at diferent locations
- Replication provides redundancy and can help improve performance
### 2. Partitioning (aka Sharding)
Splitting a big database into smaller subsets called partitions so that different partitions can be allocated to different nodes

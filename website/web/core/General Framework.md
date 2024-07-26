## General Framework to tackle system design
4 step framework

## Understand the problem and establish design scope
* Problem presented would usually be vague, therefore we should ask clarifying questions to establish scope of design. Do not jump right into solutioning.
* Clarify the requirements. Who are the users and what features need to be built
* Come up with a list of top priority features to build and have the interviewer agree to it.
* Non functional requirements are important as well. Focus on scalability and performance. Other areas include security, consistency, accuracy, freshness, availability, reliability.
* might have to do some back of the envelope calculation and estimation on scale.
* Identify core entities of the system.  These are the core entities that your API will exchange and that your system will persist in a Data Model. In the actual interview, this is as simple as jotting down a bulleted list and explaining this is your first draft to the interviewer.
* Design api / system interface. REST, Graphql or Wire protocol. GRPC?.

## Propose high level design and get buy in 
* primary goal is to design an architecture that satisfies the API you've designed and, thus, the requirements you've identified. In most cases, you can even go one-by-one through your API endpoints and build up your design sequentially to satisfy each one.
* 
## Design deep dive
* A
## Wrap up / Summary

## Extras
1. Functional requirements
    - Core features
    - If its product based, user should be able do X, Y, Z (Do not have too many reqs)
2. Non Functional Requirements
    - Do you want a consistent, or highly available system? This could influence design choices
    - Need to know the typical examples of strongly consistent systems, and highly available systems
    - What scale? How many DAU?
    - Any latency requirements? Define a specific latency range rather than "low latency".
    - Durability: how important is it that data in system is not lost? social media vs banking systems
    - Security: How secure should data be? data protection? access controls? regulatory requirements?
    - Fault Tolerance: How well does the system need to handle failures? Consider redundancy, failover, and recovery mechanisms.
3. Capacity estimations 
    - Compute QPS, size, users, db size only if it directly influence the design. Back of the envelope may not always be necessary.
    - In most scenarios, you can assume to be dealing with a large distributed systems.
    - Explain to the interviewer that you would like to skip on estimations upfront and that you will do math while designing when/if necessary. When would it be necessary? Imagine you are designing a TopK system for trending topics in FB posts. You would want to estimate the number of topics you would expect to see, as this will influence whether you can use a single instance of a data structure like a min-heap or if you need to shard it across multiple instances, which will have a big impact on your design.
4. Scaling
    - Vertical scaling (More powerful machines) vs Horizontal scaling (adding more machines)
    - Vertical scaling requires significantly less incremental complexity, if workload can be estimated to scale vertically, it can often be a better solution than horizontal scaling.
    - When horizontally scaling, one needs to contend with distribution of work, data and state across systems.
    - When encountering a scaling problem, do not always choose to go for horizontal scaling. Need to understand the design.
    - **Work distribution**. The challenge of getting the right work to the right machine. This is often done by a load balancer, which will choose the node from a group to use for the incoming request. (least connection, utilization based, simple round robin). distribution tries to keep load on a system as even as possible. This is to get the most of horizontal scaling.
    - **Data distribution**. For some systems, this implies keeping data in-memory on the node that is processing the request. More often, it implies a shared database across all nodes. Look for ways to partition the data such that one node can access all the data that it needs without talking to another node. If you have to talk to other nodes, keep the number small. (*fan-out concept*). if fan-out is large, it can be problematic due to alot of network traffic and sensitivity of any failure in each connection, and suffers from tail end latency if final result depends on all response.
    > If your system design problem involves geography, there's a good chance you have the option to partition by some sort of REGION_ID. For many systems that involve physical locations, this is a great way to scale because for many problems a given user will only be concerned with data in or around a particular location (e.g. a user in the US doesn't need to know about data in Europe).
    Horizontal scaling on data introduces synchronization challenges. Either reading or writing to shared database, or some servers are keeping copies of redundant data. This means that one have to take care of race condition and consistency challenges. Most database systems are built to resolve some of these problems by using transactions (all or nothing).
    In other cases, a **distributed lock** may be required.
5. Consistency
    - discuss the consistency that you want the system to take. **Strongly Consistent** (all subsequent reads will reflect a write) or **Eventually Consistent** (a period where read data might be stale, but eventually data will be consistent across all nodes).
    - It is not a MUST to have one type of consistency. Different parts of a system can adopt different consistency patterns.
    - notion of consistency applies to every layer of the design. Even if you use a strongly consistent database, if you have cache that have a ttl expiration policy, reads through that cache will be eventually consistent.
6. Locking

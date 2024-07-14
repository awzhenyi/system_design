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



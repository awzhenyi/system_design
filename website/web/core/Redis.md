# Redis

- Single Threaded
- key value store
- accept commands like SET, GET, INCR, XADD

Scaling Reads

Scaling Writes
- Each node takes up different range of slots. eg main1(0 - 100), main2(0 - 200), ... Each main knows the existence of each other and which slots each of them have.

- Sharding based on the keys is the only way to scale redis.
- Hot key problem, one of the node experiences uneven distribution of traffic. How to solve? append random number to key to distribute

-keyspace is key to scaling redis.

Common Usage
- cache. check cache before query db. when key evicts / expires then persist. Expiration policy -> ttl 60s, LRU

- rate limiter. use atomic command INCR to check how many request have been made to expensive service. Once request is made, expires in 60s / the calculated time range.

- redis primitive STREAM. ordered list of items, id = timestamp of insertion. Each item can have key-value

- async job queue. Job/Items that needs to be processed goes into the stream. There is a consumer group that points to the next item to be processed in stream, and allocates it to workers. The worker has claim to this item. If the worker fails, the consumer group can reclaim the item and allocate it to another worker. Worker give heartbeat to consumer group to let it know it is still alive so consumer group do not reclaim back the job. Consumer group offers at least once gaurantee. Why? Imagine worker lose connection but still processing, consumer group reclaims and allocate to worker 2.

- sorted set, heap
- geospatial data
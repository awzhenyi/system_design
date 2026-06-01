---
sidebar_position: 5
---
# Deep dive areas

## 1. Handle frequent driver location updates and efficient proximity searches on location data
### High Frequency of writes
- potentially millions of writes per second if we assume millions of drivers and location updates happen every 5 seconds.
- standard databases like postgresql or dynamoDB will not be able to handle this load / too expensive to scale

### Query Efficiency
- Without any optimizations, to query a table based on lat/long we would need to perform a full table scan, calculating the distance between each driver's location and the rider's location. This would be extremely inefficient, especially with millions of drivers. 
- Even with indexing on lat/long columns, traditional B-tree indexes are not well-suited for multi-dimensional data like geographical coordinates, leading to suboptimal query performance for proximity searches.

### Solution
#### Batch Processing with Specialized Geospatial Database
- updates are aggregated over a short interval and then batch written into database
- specialized geospatial database (Postgre with GIS plugin) with appropriate indexing to support efficient location queries
- geospatial database uses specialized data structures like quad trees and r-trees to index locations
- batch processing introduces delay, which may be undesirable since location data in database is not the most up to date, leading to suboptimal matching of driver to rider.

#### Real Time In-Memory Geospatial Data Store
- use redis which support geospatial data types and commands.
- it uses geohashing to encode long/lat coordinates to a string key which is then indexed using a sorted set
- geospatial commands like `geoadd` and `geosearch` for querying nearby locations
- durability could be a cause of concern when using redis, since there is risk of data loss
- could be mitigated with redis persistence / sentinel
- but data loss in this system could have minimal impact due to updates coming every 5 seconds. 
 
## 2. Preventing system overload with frequent driver location updates
- high frequency location updates from drivers could strain location service.

### Solution

#### Horizontal Scaling of Location Service
- more instances to cope with scale of api calls 

#### Adaptive Location Update Intervals
- not every driver has to send updates every 5 seconds.
- sensors and location data on phone to check for stationary or drivers that are moving slowly -> have them update their location less often
- algorithm to decide update frequency could be complicated

## 3. Ensuring Consistency - Preventing Multiple Ride Requests From Being Sent To Same Driver
- consistency in ride matching is a non functional requirement
- one rider requests one driver and only one driver receives the request at a time

### Solution

#### Database Status Update With Timeout Handling
- move lock to database, make use of db transactional capabilities to ensure only one instance can lock the ride at a time.
- update status accordingly based on driver accept or deny
- set timeout mechanism in ride service to release the lock if the driver did not respond within the stipulated time period
- ride service might crash and hold the lock indefinitely -> solution is to have regular cron job scheduled to sweep for expired locks. This adds unnecessary complexity.

#### Distributed Lock with TTL (Redis)
- use in memory datastore like redis
- when a ride request is sent to a driver, creates a lock with a unique identifier like driverId and TTL for the stipulated time window given to drivers to accept/deny the ride request.
- Ride Matching Service will try to acquire the lock based on driverId. When having the lock, means no other instances of ride matching service can try to acquire the same driver lock.
- If accept, ride matching service sets the status to `accepted` in the database and release the lock
- If driver do not accept the ride, lock will automatically release after the TTL.
- challenges of availability and durability of redis for locking, possiblity of lost / corrupted locks.

## 4. No Ride Requests Dropped During Peak Periods
- System might get higher volume of requests during peak periods, which could lead to dropped requests

### Solutions

#### Dynamic Scaling Queue
- requests from ride service gets put into a queue for Ride Matching Service to process
- If queue grows too large, dynamically scale the Ride Matching Service
- queues can be partitioned via geographic regions for more efficiency
- kafka is a good option for a distributed message queue
- FIFO queue has issues 

## 5. Further Scaling
- How can we further improve the scalability of the systems

### Solutions

#### Geosharding with Read Replicas
- shard data geographically and use read replicas to increase read throughput
- sharding increases the complexity and the need to manage multiple servers
- data can be distributed evenly across shard with consistent hashing




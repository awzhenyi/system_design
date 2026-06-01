# Seven Strategies to Scale Databases

## 1. Indexes

1. Index, Clustered, NonClustered, Composite
- **Cluster Index** - reorganises the data, only can have 1. Primary key is usually clustered index. faster than non clustered index.
- **NonClustered Index** - a pointer / reference to where data is stored, can have multiple.
- **Composite Index** - 

2. Advantages vs Disadvantages of creating indexes
- improves querying speed, but takes up more space, and also slower on upserts because the database and the indexes have to be updated.
- improves speed of join. 

3. How to choose which column to index?
- see query patterns, things is the where clause should preferably be indexed to prevent full table scan.
- joins on the column should be indexed.

## 2. Materialized Views

## 3. Denormalization
Reduce complex join but putting same data in different tables.
## 4. Vertical Scaling
Improve cpu, ram 
## 5. Database Caching
redis/memcached/application
challenges - cache invalidation
## 6. Replication
asynchronous or synchronous replication
## 7. Sharding

# Scaling Reads

# Scaling Writes

---
sidebar_position: 2
---

## Architecture of Redis

### Overview
Redis (Remote Dictionary Server) is an in-memory data structure store that can be used as a database, cache, message broker, and queue. Its architecture is designed for high performance and flexibility.

### Core Components

#### 1. In-Memory Data Store
- Primary storage is in RAM for maximum performance
- Data structures are stored in memory-mapped files
- Supports persistence through RDB snapshots and AOF logs

#### 2. Single-Threaded Event Loop
- Uses a single-threaded event loop for command processing
- Non-blocking I/O operations
- Efficient handling of multiple client connections
- Commands are executed atomically

#### 3. Data Structures
- Strings
- Lists
- Sets
- Sorted Sets
- Hashes
- Bitmaps
- HyperLogLogs
- Geospatial indexes

### Persistence Mechanisms

#### 1. RDB (Redis Database)
- Point-in-time snapshots of the dataset
- Binary format for compact storage
- Fork-based persistence
- Configurable save points

#### 2. AOF (Append Only File)
- Log of all write operations
- Configurable fsync policies
- Automatic rewrite for log compaction
- Better durability guarantees

### Replication Architecture

#### 1. Master-Slave Replication
- Asynchronous replication
- One master, multiple slaves
- Automatic failover support
- Partial resynchronization

#### 2. Redis Sentinel
- High availability solution
- Automatic failover
- Configuration provider
- Monitoring of master and slave instances

### Clustering

#### 1. Redis Cluster
- Automatic sharding
- No proxy required
- 16384 hash slots
- Master-slave model within cluster
- Automatic failover
- Support for multiple key operations

### Security Features

#### 1. Authentication
- Password protection
- ACL (Access Control List) support
- TLS/SSL encryption

#### 2. Network Security
- Bind to specific interfaces
- Protected mode
- Connection encryption

### Performance Optimizations

#### 1. Memory Management
- Configurable maxmemory
- Eviction policies
- Memory fragmentation handling
- Active defragmentation

#### 2. Client Connection Handling
- Connection pooling
- Pipelining support
- Pub/Sub messaging
- Lua scripting

### Monitoring and Management

#### 1. Built-in Tools
- INFO command
- MONITOR command
- SLOWLOG
- Memory usage statistics

#### 2. External Tools
- Redis-cli
- Redis-benchmark
- Redis-check-aof
- Redis-check-rdb

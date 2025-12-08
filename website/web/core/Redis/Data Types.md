# Redis Data Types

Redis supports various data types that can be used to solve different problems efficiently. This document covers the most commonly used data types with examples and use cases.

## Strings

Strings are the most basic Redis data type, capable of storing text, binary data, or numbers.

### Basic Operations
```redis
# Set and Get
SET user:1:name "John Doe"
GET user:1:name

# Increment/Decrement
SET counter 0
INCR counter
DECR counter

# Multiple operations
MSET user:1:name "John" user:1:email "john@example.com"
MGET user:1:name user:1:email

# String manipulation
APPEND user:1:name " Smith"
STRLEN user:1:name
```

### Use Cases
1. **Caching**
   - Store serialized objects
   - Cache API responses
   - Session data

2. **Counters**
   - Page views
   - Rate limiting
   - Real-time analytics

3. **Bit Operations**
   - Feature flags
   - User presence
   - Bloom filters

## Hashes

Hashes are maps between string fields and string values, perfect for representing objects.

### Basic Operations
```redis
# Set and Get fields
HSET user:1 name "John" email "john@example.com" age "30"
HGET user:1 name
HGETALL user:1

# Multiple operations
HMSET user:2 name "Jane" email "jane@example.com" age "25"
HMGET user:2 name email

# Field operations
HEXISTS user:1 email
HDEL user:1 age
HINCRBY user:1 age 1
```

### Use Cases
1. **User Profiles**
   - Store user attributes
   - Update specific fields
   - Cache user data

2. **Configuration Storage**
   - Application settings
   - Feature toggles
   - Environment variables

3. **Shopping Carts**
   - Product quantities
   - Cart metadata
   - User preferences

## Sorted Sets

Sorted Sets are collections of unique elements where each element is associated with a score.

### Basic Operations
```redis
# Add and retrieve elements
ZADD leaderboard 100 "player1" 200 "player2" 300 "player3"
ZRANGE leaderboard 0 -1
ZRANGE leaderboard 0 -1 WITHSCORES

# Score-based operations
ZINCRBY leaderboard 50 "player1"
ZSCORE leaderboard "player1"
ZRANK leaderboard "player1"

# Range queries
ZRANGEBYSCORE leaderboard 100 300
ZCOUNT leaderboard 100 300
```

### Use Cases
1. **Leaderboards**
   - Game scores
   - User rankings
   - Performance metrics

2. **Time Series Data**
   - Event timestamps
   - Sensor readings
   - Log entries

3. **Priority Queues**
   - Task scheduling
   - Job processing
   - Rate limiting

## GeoSpatial

Redis GeoSpatial indexes allow you to store coordinates and perform location-based queries.

> **Note:** `GEORADIUS` and `GEORADIUSBYMEMBER` are deprecated. Use `GEOSEARCH` and `GEOSEARCHSTORE` instead.

### Basic Operations
```redis
# Add locations
GEOADD cities 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"

# Get coordinates
GEOPOS cities "Palermo"

# Calculate distance
GEODIST cities "Palermo" "Catania" km

# Find nearby locations using GEOSEARCH
GEOSEARCH cities FROMLONLAT 15 37 BYRADIUS 100 km
GEOSEARCH cities FROMLONLAT 15 37 BYRADIUS 100 km WITHCOORD WITHDIST

# Search by member using GEOSEARCH
GEOSEARCH cities FROMMEMBER "Palermo" BYRADIUS 100 km

# Store search results in another key
GEOSEARCHSTORE nearby_cities cities FROMLONLAT 15 37 BYRADIUS 100 km
```

### Use Cases
1. **Location-Based Services**
   - Nearby places
   - Store finders
   - Delivery tracking

2. **Geofencing**
   - Area monitoring
   - Location-based alerts
   - Service availability

3. **Route Planning**
   - Distance calculations
   - Location clustering
   - Coverage analysis

## Best Practices

### 1. Memory Optimization
- Use appropriate data types for your use case
- Consider memory usage when choosing between data types
- Use compression for large string values

### 2. Performance Considerations
- Use pipeline for multiple operations
- Choose appropriate commands for your use case
- Consider the complexity of operations

### 3. Data Modeling
- Design keys with proper namespacing
- Use consistent naming conventions
- Consider data access patterns

## Common Patterns

### 1. Caching with Strings
```redis
# Cache with expiration
SETEX cache:key 3600 "value"

# Cache with version
SET cache:key:v1 "value"
SET cache:key:v2 "new value"
```

### 2. Object Storage with Hashes
```redis
# Store user object
HMSET user:1 name "John" email "john@example.com" age "30"

# Update specific fields
HSET user:1 age "31"
```

### 3. Leaderboard with Sorted Sets
```redis
# Update scores
ZINCRBY leaderboard 10 "player1"

# Get top 10 players
ZREVRANGE leaderboard 0 9 WITHSCORES
```

### 4. Location-Based Features
```redis
# Find nearby stores
GEOSEARCH stores:locations FROMLONLAT 15 37 BYRADIUS 10 km WITHCOORD WITHDIST

# Calculate delivery areas
GEOSEARCH stores:locations FROMMEMBER "main_store" BYRADIUS 5 km
```

## Error Handling

### 1. Type Mismatches
```redis
# Check type before operation
TYPE key
```

### 2. Missing Keys
```redis
# Use EXISTS to check
EXISTS key

# Use default values
GET key || "default"
```

### 3. Invalid Operations
```redis
# Handle errors gracefully
TRY {
    ZADD key score member
} CATCH {
    // Handle error
}
```


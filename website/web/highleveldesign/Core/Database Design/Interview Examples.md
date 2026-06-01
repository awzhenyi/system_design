# Database Design Interview Examples

## Overview

This guide provides detailed database schema designs for common system design interview scenarios. Each example includes requirements analysis, schema design, normalization decisions, indexing strategies, and tradeoff discussions.

## Example 1: URL Shortening Service (Bit.ly)

### Requirements

**Functional:**
- Shorten long URLs
- Redirect short URLs to long URLs
- Track click statistics
- Support custom short URLs (optional)
- URL expiration (optional)

**Non-Functional:**
- High availability
- Low latency redirects (`<100ms`)
- Handle billions of URLs
- Support high read/write ratio (100:1)

### Schema Design

```sql
-- Core URLs table
CREATE TABLE urls (
    short_url VARCHAR(10) PRIMARY KEY,  -- Base62 encoded
    long_url TEXT NOT NULL,
    user_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    is_active BOOLEAN DEFAULT TRUE,
    click_count BIGINT DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Users table (if user accounts supported)
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Click tracking (for detailed analytics)
CREATE TABLE clicks (
    click_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_url VARCHAR(10),
    ip_address VARCHAR(45),
    user_agent TEXT,
    referer TEXT,
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    country VARCHAR(2),
    FOREIGN KEY (short_url) REFERENCES urls(short_url)
);

-- Indexes
CREATE INDEX idx_urls_user ON urls(user_id);
CREATE INDEX idx_urls_active ON urls(is_active, expires_at);
CREATE INDEX idx_clicks_short_url ON clicks(short_url, clicked_at);
CREATE INDEX idx_clicks_date ON clicks(clicked_at);
```

### Design Decisions

**1. Short URL as Primary Key**
- Natural key (business identifier)
- No need for surrogate key
- Direct lookup without join

**2. Denormalized Click Count**
- Pre-computed aggregate in `urls` table
- Avoids COUNT(*) on clicks table
- Updated on each click (eventual consistency acceptable)

**3. Separate Clicks Table**
- Detailed analytics separate from core functionality
- Can be archived/partitioned by date
- Doesn't slow down redirects

**4. Indexing Strategy**
- Primary key on short_url (automatic)
- Index on user_id for user's URLs
- Index on is_active + expires_at for cleanup
- Index on clicks for analytics queries

### Tradeoffs

**Normalization:**
- Click count denormalized for performance
- Acceptable: Updates are infrequent (only on clicks)

**Consistency:**
- Click count may be slightly stale
- Acceptable: Analytics don't need real-time accuracy

**Scaling Considerations:**
- Partition clicks table by date
- Use caching (Redis) for hot URLs
- Consider sharding by short_url hash

### Query Patterns

```sql
-- Redirect (most common, must be fast)
SELECT long_url FROM urls 
WHERE short_url = ? AND is_active = TRUE 
AND (expires_at IS NULL OR expires_at > NOW());

-- User's URLs
SELECT * FROM urls 
WHERE user_id = ? 
ORDER BY created_at DESC;

-- Analytics (can be slow, async)
SELECT 
    DATE(clicked_at) as date,
    COUNT(*) as clicks
FROM clicks
WHERE short_url = ?
GROUP BY DATE(clicked_at)
ORDER BY date DESC;
```

## Example 2: Social Media Feed (Twitter)

### Requirements

**Functional:**
- Users can post tweets
- Users can follow other users
- Display home feed (tweets from followed users)
- Display user timeline (user's own tweets)
- Like and retweet tweets
- Reply to tweets

**Non-Functional:**
- Very high read volume (1000:1 read/write)
- Low latency feeds (`<200ms`)
- Handle millions of users
- Real-time updates

### Schema Design

```sql
-- Users
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    follower_count INT DEFAULT 0,  -- Denormalized
    following_count INT DEFAULT 0,  -- Denormalized
    tweet_count INT DEFAULT 0,     -- Denormalized
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tweets
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    reply_to_tweet_id BIGINT NULL,  -- For replies
    retweet_of_tweet_id BIGINT NULL, -- For retweets
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    like_count INT DEFAULT 0,       -- Denormalized
    retweet_count INT DEFAULT 0,    -- Denormalized
    reply_count INT DEFAULT 0,      -- Denormalized
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (reply_to_tweet_id) REFERENCES tweets(tweet_id),
    FOREIGN KEY (retweet_of_tweet_id) REFERENCES tweets(tweet_id)
);

-- Follows relationship
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    FOREIGN KEY (follower_id) REFERENCES users(user_id),
    FOREIGN KEY (followee_id) REFERENCES users(user_id),
    CHECK (follower_id != followee_id)
);

-- Likes
CREATE TABLE likes (
    user_id BIGINT,
    tweet_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id)
);

-- Retweets
CREATE TABLE retweets (
    user_id BIGINT,
    tweet_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id)
);

-- Indexes
CREATE INDEX idx_tweets_user_created ON tweets(user_id, created_at DESC);
CREATE INDEX idx_tweets_created ON tweets(created_at DESC);
CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_followee ON follows(followee_id);
CREATE INDEX idx_likes_tweet ON likes(tweet_id);
CREATE INDEX idx_retweets_tweet ON retweets(tweet_id);
```

### Design Decisions

**1. Denormalized Counts**
- Like/retweet/reply counts in tweets table
- Follower/following counts in users table
- Updated asynchronously to avoid write bottlenecks

**2. Separate Likes/Retweets Tables**
- Many-to-many relationship
- Allows querying who liked/retweeted
- Indexed for fast lookups

**3. Feed Generation Strategy**

**Option A: Fan-out on Write (Push Model)**
```sql
-- Write-optimized feed table
CREATE TABLE user_feeds (
    user_id BIGINT,
    tweet_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (tweet_id) REFERENCES tweets(tweet_id)
);

-- On new tweet, insert into all followers' feeds
-- Fast reads, slower writes
```

**Option B: Fan-out on Read (Pull Model)**
```sql
-- Query tweets from followed users on read
SELECT t.* 
FROM tweets t
JOIN follows f ON t.user_id = f.followee_id
WHERE f.follower_id = ?
ORDER BY t.created_at DESC
LIMIT 20;
-- Slower reads, faster writes
```

**Hybrid Approach:**
- Push for active users (frequent posters)
- Pull for inactive users (rare posters)
- Best of both worlds

### Tradeoffs

**Normalization:**
- Counts denormalized for performance
- Acceptable: Slight delay in count updates

**Consistency:**
- Counts eventually consistent
- Real-time counts not critical for feeds

**Scaling:**
- Partition tweets by user_id or date
- Use caching for hot feeds
- Consider read replicas

## Example 3: E-commerce Platform

### Requirements

**Functional:**
- Product catalog
- Shopping cart
- Order management
- Inventory tracking
- User accounts
- Payment processing
- Reviews and ratings

**Non-Functional:**
- Handle high traffic
- Inventory consistency critical
- Order history important
- Support search and filtering

### Schema Design

```sql
-- Users
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Categories
CREATE TABLE categories (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    parent_category_id INT NULL,
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id)
);

-- Products
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id INT,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    sku VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

-- Shopping Cart
CREATE TABLE cart_items (
    cart_item_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY unique_user_product (user_id, product_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Orders
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL,  -- pending, processing, shipped, delivered, cancelled
    shipping_address TEXT,
    payment_status VARCHAR(50),     -- pending, paid, failed, refunded
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Order Items (historical snapshot)
CREATE TABLE order_items (
    order_item_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    product_name VARCHAR(255) NOT NULL,      -- Denormalized (historical)
    product_price DECIMAL(10,2) NOT NULL,    -- Denormalized (historical)
    quantity INT NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Inventory Transactions (for audit)
CREATE TABLE inventory_transactions (
    transaction_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    quantity_change INT NOT NULL,  -- Positive for restock, negative for sale
    transaction_type VARCHAR(50),  -- sale, restock, adjustment, return
    order_id BIGINT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

-- Reviews
CREATE TABLE reviews (
    review_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY unique_user_product_review (user_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Product Ratings Summary (denormalized)
CREATE TABLE product_ratings (
    product_id BIGINT PRIMARY KEY,
    average_rating DECIMAL(3,2),
    total_reviews INT DEFAULT 0,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_stock ON products(stock_quantity);
CREATE INDEX idx_orders_user ON orders(user_id, order_date DESC);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_cart_items_user ON cart_items(user_id);
CREATE INDEX idx_reviews_product ON reviews(product_id, created_at DESC);
CREATE INDEX idx_inventory_product ON inventory_transactions(product_id, created_at);
```

### Design Decisions

**1. Historical Data in Order Items**
- Store product name and price at time of purchase
- Product prices may change
- Historical accuracy important

**2. Inventory Management**
- Stock quantity in products table (denormalized)
- Inventory transactions table for audit trail
- Can reconcile if needed

**3. Denormalized Ratings**
- Average rating and count in separate table
- Updated asynchronously
- Fast product listing queries

**4. Shopping Cart**
- Separate table, not part of orders
- Can persist across sessions
- Easy to clear on checkout

### Tradeoffs

**Consistency:**
- Inventory: Strong consistency critical (use transactions)
- Ratings: Eventual consistency acceptable

**Normalization:**
- Order items denormalized for historical accuracy
- Ratings summary denormalized for performance

**Scaling:**
- Partition orders by user_id or date
- Use read replicas for product catalog
- Cache hot products

## Example 4: Leaderboard System

### Requirements

**Functional:**
- Users can update scores
- Display top N users
- Get user's rank
- Get users around a specific user
- Multiple leaderboards (by game, time period)

**Non-Functional:**
- Real-time updates (`<100ms`)
- Handle millions of users
- High update frequency
- Low latency reads

### Schema Design

```sql
-- Users
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(100),
    created_at TIMESTAMP
);

-- Leaderboards
CREATE TABLE leaderboards (
    leaderboard_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    game_id INT,
    time_period VARCHAR(50),  -- daily, weekly, monthly, all_time
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User Scores
CREATE TABLE user_scores (
    user_id BIGINT,
    leaderboard_id INT,
    score INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, leaderboard_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (leaderboard_id) REFERENCES leaderboards(leaderboard_id)
);

-- Indexes
CREATE INDEX idx_scores_leaderboard_score ON user_scores(leaderboard_id, score DESC, user_id);
CREATE INDEX idx_scores_user ON user_scores(user_id);
```

### Design Decisions

**1. Composite Index for Ranking**
- Index on (leaderboard_id, score DESC, user_id)
- Enables efficient top N queries
- Supports rank calculation

**2. Real-time Updates**
- Update score in place
- Use transactions for consistency
- Consider Redis Sorted Sets for very high frequency

### Alternative: Redis Sorted Sets

For extremely high-frequency updates:

```python
# Redis Sorted Set
redis.zadd('leaderboard:1', {user_id: score})
redis.zrevrange('leaderboard:1', 0, 9)  # Top 10
redis.zrevrank('leaderboard:1', user_id)  # User's rank
```

**Hybrid Approach:**
- Redis for real-time leaderboard
- Database for persistence
- Periodic sync

### Query Patterns

```sql
-- Top N users
SELECT u.*, us.score
FROM user_scores us
JOIN users u ON us.user_id = u.user_id
WHERE us.leaderboard_id = ?
ORDER BY us.score DESC
LIMIT 10;

-- User's rank
SELECT COUNT(*) + 1 as rank
FROM user_scores
WHERE leaderboard_id = ?
AND score > (
    SELECT score FROM user_scores
    WHERE user_id = ? AND leaderboard_id = ?
);

-- Users around specific user
WITH user_score AS (
    SELECT score FROM user_scores
    WHERE user_id = ? AND leaderboard_id = ?
)
SELECT u.*, us.score
FROM user_scores us
JOIN users u ON us.user_id = u.user_id
WHERE us.leaderboard_id = ?
AND us.score BETWEEN (
    (SELECT score FROM user_score) - 100
) AND (
    (SELECT score FROM user_score) + 100
)
ORDER BY us.score DESC
LIMIT 20;
```

## Example 5: Notification System

### Requirements

**Functional:**
- Send notifications to users
- Multiple notification types (email, push, SMS)
- Notification preferences
- Mark as read/unread
- Notification history

**Non-Functional:**
- High throughput
- Low latency delivery
- Handle millions of notifications
- Support different channels

### Schema Design

```sql
-- Users
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    phone VARCHAR(20)
);

-- Notification Templates
CREATE TABLE notification_templates (
    template_id INT PRIMARY KEY AUTO_INCREMENT,
    type VARCHAR(50) NOT NULL,  -- welcome, order_confirmation, etc.
    subject VARCHAR(255),
    body TEXT,
    channel VARCHAR(20)  -- email, push, sms
);

-- Notifications
CREATE TABLE notifications (
    notification_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    template_id INT,
    type VARCHAR(50) NOT NULL,
    title VARCHAR(255),
    message TEXT,
    channel VARCHAR(20) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',  -- pending, sent, failed, read
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    sent_at TIMESTAMP NULL,
    read_at TIMESTAMP NULL,
    metadata JSON,  -- Additional data for personalization
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (template_id) REFERENCES notification_templates(template_id)
);

-- User Notification Preferences
CREATE TABLE user_notification_preferences (
    user_id BIGINT,
    notification_type VARCHAR(50),
    channel VARCHAR(20),
    enabled BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (user_id, notification_type, channel),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Indexes
CREATE INDEX idx_notifications_user_status ON notifications(user_id, status, created_at DESC);
CREATE INDEX idx_notifications_status_created ON notifications(status, created_at);
CREATE INDEX idx_notifications_user_read ON notifications(user_id, read_at);
```

### Design Decisions

**1. Status Tracking**
- Track notification lifecycle
- Enables retry logic
- Supports analytics

**2. Template System**
- Reusable notification templates
- Easy to update content
- Supports personalization

**3. Preferences**
- User control over notifications
- Per-channel preferences
- Respect user choices

**4. JSON Metadata**
- Flexible additional data
- Personalization support
- Extensible without schema changes

### Scaling Considerations

**High Volume:**
- Partition notifications by user_id or date
- Use message queue for async processing
- Batch sending for efficiency

**Query Patterns:**
```sql
-- Get unread notifications for user
SELECT * FROM notifications
WHERE user_id = ? AND status = 'sent' AND read_at IS NULL
ORDER BY created_at DESC
LIMIT 20;

-- Get pending notifications to send
SELECT * FROM notifications
WHERE status = 'pending' AND created_at < NOW() - INTERVAL 1 MINUTE
ORDER BY created_at
LIMIT 1000;
```

## Interview Tips

### 1. Start with Requirements
- Clarify functional requirements
- Understand non-functional requirements
- Ask about scale and performance

### 2. Design Incrementally
- Start with core entities
- Add relationships
- Consider edge cases

### 3. Discuss Tradeoffs
- Explain normalization decisions
- Justify denormalization
- Discuss consistency choices

### 4. Consider Scaling
- Mention partitioning strategies
- Discuss caching
- Consider read replicas

### 5. Think About Queries
- Design indexes based on queries
- Consider query patterns
- Optimize for common operations

## Next Steps

- Review [Best Practices](./Best%20Practices.md) for production considerations
- Study [Tradeoffs](./Tradeoffs.md) for decision frameworks
- Understand [Schema Design Patterns](./Schema%20Design%20Patterns.md) for reusable solutions


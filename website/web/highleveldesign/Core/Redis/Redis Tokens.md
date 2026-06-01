# Redis Tokens

## What Are Redis Tokens?

Redis tokens are unique, lightweight data items (often implemented as keys, values, or elements in a Redis data structure) used to represent limited resources or permissions in distributed systems. They are commonly used to control access, manage quotas, or represent inventory in real-time applications.

## How Redis Tokens Work

- **Token Generation:** A fixed number of tokens are created in Redis, each representing a unit of a resource (e.g., one item in stock).
- **Token Consumption:** When a user requests a resource, the system attempts to consume (remove or decrement) a token from Redis.
- **Atomicity:** Redis operations (like `DECR`, `LPOP`, or `SPOP`) are atomic, ensuring that no two users can consume the same token simultaneously.
- **Exhaustion:** Once all tokens are consumed, no further requests can be fulfilled, preventing over-allocation.

## Example Scenario: Flash Sale System

### Problem
In a flash sale, a limited number of items are available for purchase. High concurrency can lead to overselling if not managed properly.

### Solution Using Redis Tokens
- **Initialization:** Before the sale, push N tokens (e.g., item IDs or placeholders) into a Redis list, where N is the available stock.
- **Purchase Attempt:**
  - When a customer tries to buy, the system attempts to `LPOP` a token from the list.
  - If successful, the purchase proceeds.
  - If the list is empty, the item is sold out, and the purchase is denied.
- **Guarantee:** No more than N purchases can succeed, ensuring stock is not oversold.

#### Example Redis Commands
```redis
# Initialize 100 tokens for 100 items
LPUSH flash_sale_stock token1 token2 ... token100

# Attempt to purchase
LPOP flash_sale_stock
```

## Other Use Cases
- **API Rate Limiting:** Tokens represent allowed requests per time window.
- **Distributed Locks:** Tokens act as lock permits for critical sections.
- **Resource Quotas:** Limit concurrent access to services or features.

## Benefits
- **Atomicity:** Redis ensures token operations are atomic, preventing race conditions.
- **Performance:** In-memory operations are extremely fast, suitable for high-concurrency scenarios.
- **Simplicity:** Easy to implement and reason about.

## Summary
Redis tokens are a powerful pattern for managing limited resources in distributed systems. By leveraging Redis's atomic operations, you can ensure fair allocation and prevent overuse or overselling, especially in high-demand scenarios like flash sales.

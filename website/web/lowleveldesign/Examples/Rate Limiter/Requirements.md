# Rate Limiter Requirements

## Problem Statement

Build an in-memory rate limiter for an API gateway.

The system receives configuration from an external service that provides rate limiting rules per endpoint. Each endpoint can have its own limit and can use a specific rate limiting algorithm.

## Example Configuration

```json
{
  "endpoint": "/search",
  "algorithm": "TokenBucket",
  "algoConfig": {
    "capacity": 1000,
    "refillRatePerSecond": 10
  }
}
```

This configuration allows bursts of up to `1000` requests and refills at `10` requests per second.

## Functional Requirements

1. Configuration is provided at startup and loaded once.
2. The system receives requests containing:
   - `clientId: string`
   - `endpoint: string`
3. Each endpoint has a configuration specifying:
   - Algorithm to use, such as `TokenBucket` or `SlidingWindowLog`
   - Algorithm-specific parameters, such as `capacity` and `refillRatePerSecond` for token bucket
4. The system enforces rate limits by checking the `clientId` against the endpoint's configuration.
5. The system returns a structured result:
   - `allowed: boolean`
   - `remaining: int`
   - `retryAfterMs: long | null`
6. If an endpoint has no configuration, the system uses a default limit.

## Out of Scope

- Distributed rate limiting with Redis or cross-node coordination
- Dynamic configuration updates
- Metrics and monitoring
- Config validation beyond basic checks

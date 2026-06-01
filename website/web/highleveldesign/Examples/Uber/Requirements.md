---
sidebar_position: 1
---

# Requirements

## Functional
1. Rider able to get a fare estimate based on current location and destination
2. Rider able to book a ride based on this estimate
3. Driver able to accept a ride and get to passenger location

## Non-Functional
1. Strong consistency for driver 
    a. accepting only one ride at a time
    b. no more than 1 driver can accept the same ride
2. low latency matching (to reduce waiting time for both drivers and riders)
3. able to handle high throughput especially during peak periods (100k requests per second)


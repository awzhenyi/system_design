---
sidebar_position: 4
---

Build incrementally from the functional requirements and apis listed.

# User get estimated fares

1. Rider Client (ios, android)
2. API gateway: a layer between rider client and backend services
3. Ride Service -> a service which handles estimated fare
    - interacts with 3rd party mapping api
    - complex matching algorithm
4. Ride Database 
    - responsible for storing fare

## Interaction
1. Rider enters pickup and destination into the app and it makes a POST request via /fare
2. API gateway receives the request (does authentication, authorization, rate limiting) and routes it to ride service
3. ride service makes a request to third party mapping api to get distance, etc and applies the pricing logic with the information
4. creates a new Fare object in db
5. returns the Fare object to the api gateway which forwards it back to the client app.

# User request rides
1. 

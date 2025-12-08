---
sidebar_position: 3
---

1. POST: /fare -> returns a Fare
```
req: {
    pickup_location: coordinates
    destination: coordinates
}

response : {
    fare: {
        id,
        cost
    }
}
```

2. POST: /rides
```
req: {
    fareId
}
```

3. PATCH: /rides/{rideId}
```
req: {
    {
        op: replace
        path: /status
        value: accept/deny/cancel
    }
}
```

4. POST /drivers/locations
```
req: {
    latitude,
    longitude
}
```
* note that driverId should not be passed in body, instead it should be in session cookie / JWT.
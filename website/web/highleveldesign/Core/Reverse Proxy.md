# Reverse Proxy
- acts as an intermediary between client requests and web servers / services.
- forwards client requests to the appropriate servers and returns the response from the server back to the client.
- provides benefits such as 
    - increased security (hides backend servers), 
    - load distribution, 
    - ssl termination (terminate HTTPS traffic from clients, relieving your upstream web and application servers of the computational load of SSL/TLS encryption.)
    - compression
    - caching

- common load balancers include HAProxy, Nginx.
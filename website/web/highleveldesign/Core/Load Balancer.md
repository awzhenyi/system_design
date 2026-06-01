# Load Balancer

* Web apps are deployed on machines and hosted on servers. These hardware machines have finite resources such as memory, network connection and processing power.
* Increase in traffic can prevent applications from serving requests once it hits its capacity.
* Complements horizontal scaling, in determining how/which servers to send requests to.
* Good load balancer will evenly distribute requests to all servers.
* Can operate at higher level (HTTP) vs lower level (TCP). Can be implemented in hardware as well.
* typically use HAProxy or Nginx
* Cloud solutions offer out of the box load balancer, eg AWS Elastic Load Balancer.
* For microservice architecture, important to have load balancer in front of every service so each service can be scaled independently.
* Some scaling issues cant be solved simply by load balancer. Eg: Database performance issue, algorithmic complexity. => use job queue instead.

Advantages
1. Scalability -> scale on demand by removing or adding backend servers.
2. Reliability -> provide redundancy and reduce downtime by automatically detecting and replacing unhealthy servers.
3. Improve average response time by distributing load.

Considerations
1. As scale increases, load balancer itself can be bottleneck / single point of failure -> use multiple load balancers to gaurantee availability. DNS round robin can be used to balance traffic across multiple load balancer.
2. same user request can be served from different backend servers unless the load balancer is configured otherwise. This could be problematic for application that rely on session data that are not shared among different servers.

Load Balancing algorithms
1. Round Robin
2. Least Connection
3. Consistent Hashing

- common load balancer includes nginx.


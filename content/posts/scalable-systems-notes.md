+++
title = 'Scalable Systems Notes'
date = 2024-01-20T12:37:25-08:00
draft = false
+++

## Basics

- Scalability is the property of a system to handle a growing amount of work by adding resources.
- Replication and Optimization
   - Replication adds more resources (web servers, app servers, database replicas, etc).
   - Optimization makes the most of the resources we have (more efficient algorithms, faster programming languages, adding database indexes, etc).
- The goal is to have systems that scale exponentially while costs grow linearly (hyperscale systems).

## Distributed Architectures

**Starter System**

- Consists of a client tier, an application service tier, and a database tier.

**Scaling Up**

- Means upgrading a server with more CPUs and memory. It's simple and can go a long way.

**Scaling Out**

- Relies on the ability to run multiple copies of a service on multiple servers. You will need:
   - Load balancers are reverse proxies that process user requests with the aim of keeping all resources equally busy.
   - Stateless services: user sessions need to be stored outside of the replicated application servers so that if a server fails, the load balancer can simply redirect the request to a different server.
   - A scaled out system involves the client tier, a load balancer, multiple application servers, a separate session store, and the database tier.
   
**Distributed Cache**

- The cache can store commonly accessed database results in memory to reduce hits to the database so you effectively buy extra database capacity.
- A scaled out system can replace the session store with a caching system since sessions can be stored in the cache.
- Caveats are stale data and managing cache invalidation so it's best to cache data that rarely changes.

**Distributed Databases**

- **Distributed SQL stores** store data across multiple servers but logically appear to the application as a single database.
- **Distributed NoSQL stores** are usually built from the ground up with distributing data across multiple nodes in mind. Leading products are Cassandra, MongoDB, and Neo4J.
- At this point, the architecture now includes the client tier, a load balancer, distributed application servers, distributed cache, and distributed databases. This makes for a relatively highly available system since another server can always pick up the slack if a particular server fails.

**Services**

- You can also break up the application into multiple independent services (or microservices) so you can scale resources on each service based on demand.
- One example is having one service for web users and another for mobile where you can increase or decrease resources for each as needed.

**Increasing Responsiveness**

- Not all data needs to persist immediately before returning to the user.
- You can send data to a distributed queueing platform where a *producer* writes to queue while a *consumer* removes the message from the queue which can then process the request before persisting it in the database.
- Use cases for this would be for processing comment sentiment analysis or text spam classification since the user does not need to wait around on the initial request for these background processes finish.

## Distributed Architectures

**Horizontal Scaling**

- The simplest approach to easily add new processing capacity is by deploying multiple instances of stateless servers with a load balancer distributing requests across these instances.
- Horizontal scaling reduces single points of failure and increases availability.

**Load Balancing**

- Load balancers can act on layer 4 (network) or layer 7 (application).
- Network-level load balancers are the simplest and fastest since they operate on individual TCP or UDP packets and provide few features.
- Application-level load balancers reassemble the complete HTTP request and base their routing decisions on HTTP header values or the actual contents of a message.
    - For example, a load balancer can be configured to send all POST requests to a subset of services.
    - Are slower than network-level load balancers but provides richer capabilities.

**Load Distribution Policies**

- Round robin.
- Least connections.
- HTTP header field: X-Client-Location:US,Seattle can be routed to specific servers, for example.
- HTTP operation: based on the HTTP verb (GET, POST, etc).
- Services can be allocated weights so that if a server has twice the capability, it can be allocated a weight of 2 whereas the less capable server can have a weight of 1.

**Health Monitoring**

- A load balancer will periodically send pings and attempt connections to test the health of each service.
- If a service is unresponsive, it will be removed from the pool until it becomes available again.

**Elasticity**

- Elasticity is the capability of an application to automatically provision capacity in response to an increase in requests.
- Scaling policies can be configured based on the available metrics (CPU utilization, etc) of the monitoring system (example: AWS Auto Scaling groups) or capacity can be scheduled.

**Session Affinity**

- Otherwise called sticky sessions for stateful services where a load balancer sends all requests from the same client to the same service instance.
- Can be problematic for highly scalable systems since if a service goes down, it will be difficult to recreate the user's state.

## Distributed Caching

**Application Caching**

- Requires business logic to stores and access precomputed results into a distributed cache.
- Best for data that changes infrequently or is expensive to generate to relieve databases of heavy read traffic.
- Two dominant technologies in this area are memcached and Redis.
- Use cases include user sessions and database query results.
- A request to the cache that yields results is called a *cache hit* and is otherwise a *cache miss*.
- Cache access requires a key to associate with the results and a *time to live* (TTL) value is created with the object that tells the cache when to evict a key-value pair in seconds.
- Application-level caching is also known as the *cache-aside pattern* since the code can bypass the data storage systems entirely if the results are in the cache.
- Other caching patterns:
    - Read-through: The application only access the cache for data. If the data is not available, a loader accesses the data systems and loads the results in the cache for the application to access.
    - Write-through: The application always writes updates to the cache while a writer is invoked to write the new cache values to the database. When the database is updated, the application can complete the request.
    - Write-behind: Like write-through but the application does not wait for the value to be written to the database. Also known as a write-back cache and is internally used by most database engines.
- The cache-aside strategy is the primary approach for scalable systems since if the cache is unavailable, all requests are simply handled as a cache miss yet the services can still satisfy requests.

**Web Caching**

- Exploits mechanisms built into the HTTP protocol to enable caching within the infrastructure provided by the internet.
- Multiple levels of caches exist from the client's web browser cache to the local organization-based caches.
- Edge caches are also known as content delivery networks (CDNs) and are often geographically distributed to cache frequently accessed data closer to the application's users.
    - Typically stores the results of GET requests only and the cache key is the URI of the associated GET.
- Proxy caches such as Squid and Varnish are extensively deployed on the internet and is most effective for static data (images, videos, and audio) or infrequently changed data.

**Cache-Control**

- The Cache-Control HTTP header specifies how a resource should be cached.
- *no-store*: Specifies that a resource should not be cached.
- *no-cache*: Specifies that a cached resource must be revalidated with an origin server. Uses "Etag".
- *private*: Specifies that a resource can only be cached by user-specific devices such as a web browser.
- *public*: Specifies a resouce that can be cached by any proxy server.
- *max-age*: Defines the length of time in seconds a cached copy of a resource should be retained.

**Expires and Last-Modified**

- If *max-age* is not specified, the *Expires* header is checked next which has an explicit date and time as the expiration.
- The *Last-Modified* header is checked last and is used to determine the freshness lifetime of a resource based on a heuristic calculation that the cache supports.
    - A resource retention period can be calculated as the value of the *Date* header minus the value of the *Last-Modified* header divided by 10.

**Etag**

- An *Etag* can be set for a resource for a web cache to check for if the resource is still valid based on what an origin server states during revalidation.
    - If an *Etag* matches with the origin server value, a *304 (Not Modified)* response is returned with no response body needed.
    - The server can also return a *200 OK* response with a response with a response body and an *Etag* representing the latest version.

## Asynchronous Messaging

- A request to a service can be fired by a *producer* into a message queue for a *consumer* to act on at a later time so that a user does not have to wait around for all of the processes associated with a request to finish.
- A messaging system comprises the the following:
    - Message queues: store the sequence of messages.
    - Producers: sends messages to a named queue and waits for an acknowledgement from the broker before considering the send complete.
    - Consumers: retrieves messages from the queues.
        - Each message is retrieved by exactly one consumer.
        - Pull mode, also known as polling, continually polls the queue for messages.
        - In push mode, the consumer provides the broker with a callback function that should be invoked when a message is available.
        - Push mode is more efficient to prevent the broker from being flooded with requests from multiple consumers.
    - Message broker: manages one or more queues and adds messages to the queue in the order they arrive.
        - Sends an acknowledgement to producers.
        - Accepts acknowledgements from consumers to mark a message as delivered and remove it from the queue.
        - Automatic acknowledgements have the lowest latency since a message is marked as delivered as soon as a consumer takes it.
        - Manual acknowledgements are slower since a consumer must fully process the message before the broker can mark the message as delivered.

**Message Persistence**

- Message queues are typically stored in memory for lower latencies.
- Messages can be persisted to disk for data safety in the event of a crash or reboot which can be used to repopulate the queues in memory but will increase response time for send operations.

**Publish-Subscribe**

- In publish-subscribe systems, message queues are known as *topics* and can have one or more subscribers.
- New subscribers can easily be added without other changes to the system.
- Consumers can execute tasks in parallel for each message.
- The broker has more work to do in this system since it needs to keep messages available until all subscribers have consumed it.

## Serverless Processing Systems

- As an alternative to explicitly provisioning virtual processing resources on infrastructure as a service (IaaS) platforms, *serverless* platforms do not require any compute resources to be statically provisioned.
    - Examples are AWS Lambda and Google App Engine (GAE).
- Like with Autoscaling Groups, serverless platforms also manage autoscaling to provide additionaly capacity and consistent low response times.
- When request loads drop, capacity is decommissioned and no charges are incurred if there are no active requests.

## To Be Continued



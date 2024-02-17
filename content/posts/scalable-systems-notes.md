+++
title = 'Scalable Systems Notes'
date = 2024-01-20T12:37:25-08:00
draft = false
+++

## The Basics

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

## Scalable Systems

## Application Services

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
- Commonly used for implementing microservice architectures.

**Google App Engine**

- Application execution is managed by Google App Engine (GAE) and launches compute resources to match demand levels.
- Typical applications also access a managed persistent storage platform such as Firestore or Google Cloud SQL or interact with a messaging service like Google's Cloud Pub/Sub.
- GAE comes in two flavors:
   - Standard environment: more closely managed by GAE with development restrictions on language versions supported but can scale services rapidly.
   - Flexible environment: runs applications on Docker containers and gives more options for development capabilities but does not scale as rapidly as the standard environment.

**Standard Environment**

- Developers upload application code to a GAE project associated with a base project URL.
- Projects can be configured to have a minimum and a maximum number of instances with a minimum of zero being used for applications that have long periods of inactivity saving on costs.
- Since GAE is responsible for loading the runtime environment for an application, it restricts the supported versions to a small number per programming language resulting in a much faster loading of new instances especially relative to booting a virtual machine.
- The language used affects the time to load a new instance on GAE where Go may start a new instance in under a second while a JVM might load in 1-3 seconds.

**Autoscaling**

- Specified in an *app.yaml* passed to GAE when your code is uploaded.
- The default parameters can be overridden.
    - *Target CPU utilization*: CPU threshold for when more or fewer instances will be started. The range is 0.5 (50%) to 0.95 (95%). Default is 0.6 (60%).
    - *Maximum concurrent requests*: Default is 10 and the maximum is 80.
    - *Target throughput utilization*: When the number of concurrent requests for an instance reaches a value eaual to maximum concurrent requests value multiplied by the target throughput utilization, the scheduler tries to start a new instance. The range is 0.5 (50%) to 0.95 (95%). Default is 0.6 (60%).
- By default, an instance will handle 10 x 0.6 = 6 concurrent requests before a new instance is created. If these 6 or fewer requests cause the CPU utilization to go over 60%, the scheduler will also try to create a new instance.
- You can also set the *max-pending-latency* parameter to specify how long a request can wait in the pending queue before starting additional instances. The default is 30 ms.
- The number of concurrent requests an instance can handle depends on the processing time for the instance. If each request takes 100 ms to process, then each instance processes 10 requests per second. If there are 1000 instances, the maximum throughput is 1000 x 10 = 10,000 requests per second.
- To determine the parameters that result in the highest throughputs, load test a range of parameters. For example: *target_cpu_utilization*: {0.6, 0.7, 0.8} each at *max_concurrent_requests*: {10, 35, 60, 80}.
- Factor in the costs associated with each parameter combination to determine your ideal cost to performance ratio.

**AWS Lambda**

- Has many of the same capabilities as GAE with Lambda functions being tightly integrated to the AWS ecosystem such as AWS S3 and AWS CloudWatch.
- The inital invocation is known as a *cold start*.
- Unlike GAE, Lambda does not send multiple concurrent requests to the same runtime instance.
- If an instance is not immediately reutilized, Lambda *freezes* the execution environment and *thaws* it when subsequent requests arrive.
- If more requests don't arrive after a number of minutes, Lambda will deactivate the frozen instance.
- Cold start overhead can be mitigated by using *provisioned concurrency* but increased charges apply.
- Unlike GAE, vCPUs can't be specified and instead allocated in proportion to the memory specified.
- Concurrency limits actually apply to all functions in the region associated with a single AWS account so it's possible that one function can consume the burst limit and negatively affect the availability of other functions.
    - *Reserved concurrency* can mitigate this issue where each individual function can be set with a value that is less than the burst limit.
    - Requests that cannot be processed with fail with an HTTP 429 error (too many requests).

## Microservices

- Microservices solve issues that monoliths may eventually have as they grow.
    - *Code base complexity*: As the size of the application and engineering team grows, adding new features, testing, and refactoring becomes progressively more difficult. Technical debt can build up resulting in ever more fragile code and new features are rolled out at an ever-slowing pace.
    - *Scaling out*: Monoliths can be scaled out by replicating the application to multiple nodes but the entire application must be copied every time. If only a small part of the application is actually seeing a spike in traffic, system resources may be wasted hosting the entire application when only a small part of it actually needs to be scaled.
- A monolith can be decomposed into multiple independent services that communicate and coordinate with each other when necessary.
- Microservices offer the following advantages:
    - *Code base*: An individual service should not be more complex than what a small team size can build, evolve, and manage. Since a microservice is fully self-contained and is effectively a black box to the rest of the system, the team has full autonomy to choose their own development stack and data management platform. Given the narrower scope of a well-designed microservice, code complexity should be low resulting in a higher development cadence for new features.
    - *Scale out*: Microservices can be scaled out independently based on individual needs.
- Domain-driven design (DDD) is often a suitable method for identifying microservice boundaries within a monolith with the goal of creating services that are self-contained in nature.
    - Generally, code that changes together, stays together.
- A *self-contained* microservice not only refers to the code, but also means that it has its own database separate from the monolith's whether it be a logical or a physical separation.
- However, microservices are distributed by nature so excessive latencies may result in microservices frequently interacting with each other over the network.
    - Two domains may initially make sense separated but if both microservices frequently interact with each other incurring excessive communications, it may make sense to merge the two.
    - Another approach is to duplicate data across coupled microservices so data can be accessed locally to reduce response times. Duplicating data has its own trade-offs namely with additional storage capacity and the need for duplicated data across microservices to converge to a consistent state.

**Deploying Microservices**

- *Serverless platforms* is one approach where a microservice can be built to expose its API on the platform. Advantages:
    - *Deployment is simple*: just upload your executable package to the endpoint configured for your function.
    - *Pay by usage*: costs are low or zero for periods of low or no usage.
    - *Ease of scaling*: the platform scales your microservice based on your configuration parameters.
- When deploying on a serverless platform, you expose multiple endpoints that clients need to invoke which introduces complexity if a client needs to know the exact IP and port for each microservice.
    - An API gateway pattern is used to counteract this issue where the gateway essentially acts as a single entry point for all requests (refer to *facade* pattern in *Gang of Four* book for an analagous situation). This insulates clients from needing to know about the underlying architecture so that if the developer refactors the API, clients are oblivious to the change.
    - API gateways can be provided by Nginx or Kong and proxies incoming client API requests to backend microservices via pre-configured mappings between client-facing APIs and microservice IPs and ports.
    - API gateways can also handle authenication and authorization, request throttling, caching API results, and the integration of monitoring tools.
    - The API gateway can be a bottleneck. The AWS API Gateway has a 10K requests per second limit with the ability to burst up to an additional 5K requests/second. The Kong API gateway however, is stateless so it is possible to deploy multiple instances and distribute requests using a load balancer.

**Principles of Microservices**

- *Modeled around a business domain*: A collection of related services can come under a *domain*. Business domain boundaries may need rethinking in the context of coupling between microservices and the performance overheads it may introduce.
- *Hide implementation details*: The API contract should be guaranteed but the internals should be a black box which allows for the flexibility for the service team to choose their own development stack.
- *Deploy independently*: Each microservice should be deployable independently of other microservices so that a service team does not have to depend on any other team.
- *Isolate failure*: A microservice failure should not propagate to the others. The system should continue to operate, even at a reduced functionality, if a microservice fails.
- *Highly Observable*: Each service must be monitored to ensure low latencies, that error conditions are logged, and that they are behaving as expected.
- *Decentralize all the things*: Decentralize the processing of client requests that require multiple calls to downstream microservices (often called workflows). Two basic approaches are orchestration and choreography.
- *Culture of automation*: Development and DevOps tooling are essential to gain the benefits of microservices making the processes faster and more robust to make changes to the system.

**Resilience in Microservices**

- **Cascading Failures**: Microservices frequently need to communicate to satisfy a single request making them susceptible to cascading failures. These occur when downstream service response times become slow which cause back pressure in calling services and eventually, a failure in one can cause all dependent services to crash. This can often be excaerbated by clients that immediately retry the operation. The core problem is that slow services utilize system resources for requests for extended periods.
    - You can insert a growing delay between retries, a technique called exponential backoff.
    - *Fail fast pattern*: You may have 50% of requests fall under 200 ms, 95% fall under 1200 ms, and 99% faull under 3000 ms. If the API handless 200 million requests per day, this means that 1%, or 2 million requests, take greater than 3 seconds.
        - A system can fail requests that take longer than a predefined time limit (ex. > 3 seconds) to immediately release resources associated with that request.
        - You can also enable throttling or rate limiting on server if the request load exceeds some threshold and fail the request with an HTTP 503 error (Service unavailable) and maybe have the UI display a default until the service returns.
    - *Circuit breaker pattern*: If a microservice starts to throw errors, instead of failing fast and still incurring a timeout delay, it may be better to just back off immediated from sending further requests until service quality resumes.
        - One popular library in Python is CircuitBreaker where you simply decorate the external call you want to protect with @circuit (ex. `@circuit(failure_threshold=20,expected_exception=RequestException,recovery_timeout=5`).
        - The service can return a default or cached results while the circuit breaker is open.
        - Ensure circuit breaker triggers are monitored and logged for future diagnosis.
    - *Bulkhead pattern*: a damage limitation strategy.
        - If a part of a service is flooded with requests and sapping the system of resources, you may want to reserve a portion of resources to critical areas of the same service to maintain some level of functionality (ex. new order creations should still work whereas order tracking requests may not).

**References**

- *Building Microservices*, 2nd Edition (O'Reilly, 2021).

## Scalable Distributed Databases

## Scalable Database Fundamentals

**Scaling Relational Databases**

- *Scaling Up*: Migrating to more powerful hardware.
    - Downsides: Cost, Availability, Growth
    - If clients are global and low latency responses to each client is important, scaling up does not help with the issue.
- *Scaling Out*: Configure one or more nodes as replicas (secondaries) of the main database (primary). Writes are only possible to the primary but secondaries can be geographically distributed and reads are spread among the replicas.
    - Since there is a delay between the write on the primary and the replication to the secondaries, the application must be aware of the possibility of reading stale data.
- **Partitioning Data**
    - *Horizontal partitioning*: Split data based on a value of a row or a hash function on the primary key. One example is splitting data based on customer region (`WHERE region=3`).
    - *Vertical partitioning* (row splitting): Splits data by the columns in a row. A common strategy is to partition a row between static, read-only data and dynamic data.
    - A problem comes up with partitioned data where if a single request needs data from multiple nodes or join data from distributed partitioned tables potentially resulting in high levels of network traffic and required request coordination. There are strategies to mitigate this problem:
        - Define reference tables that are relatively small, rarely change, and need to be joined against frequently. These can be copied to each node so join operations cane execute locally and in parallel on each partitioned node with results to be merged by the request coordinator.
        - Use partition keys or secondary indexes in joins to allow joins to execute locally.
        - Ensure one side of the join has a highly selective filter that reduces the rows to a small collection. This can then be sent to each partitioned node and the join proceeds as in a reference table join minimizing data movement.
        - Google's Cloud Spanner distributed relational database has multiple join algorithms and will choose algorithms automatically.
- **Shared Everything Databases**: An example is Oracle's Real Applications Cluster (RAC) where you can have several replicated database engines coordinated through a clusterware layer, a shared cache among the clustered database nodes, all sharing the same data storage (database files, transaction logs, control files) over a Storage area network (SAN).

**NoSQL**

- Simplified data models that can be easily evolved.
- Proprietary query languages with limited or no support for joins.
- Native support for horizontal scaling on low-cost, commodity hardware.
- Generally designed to natively scaled horizontally using a *shared nothing* architecture with no shared state.
- Without joins, the emphasis on changes from problem domain modeling to modeling the solution domain which requires you to think about the common data access patterns the application must support.
- For reading data, this means that the data model must *prejoin* the data you need for the request, otherwise known as a denormalized data model for relational databases.
- Another way to think of it is creating a table per use case.
- NoSQL databases are also usually termed as *schemaless* databases that don't have to define the format of objects up front but with the trade-off of the application having to discover the structure of the data it reads.

**NoSQL Data Models**

- *Key-value*: Basically a hash map (ex. Redis, Oracle NoSQL).
- *Document*: Typically encoded in JSON usually organized into logical collections (ex. MongoDB, Couchbase).
- *Wide column*: Organizes data associated with a key in named columns. Essentially a two-dimensional hash map, enabling columns within a row to be uniquely identified and sorted using a column name (ex. Apache Cassandra, Google Bigtable).
- *Graph*: Used for storing and querying highly connected data and treats relationships between database objects as first-class citizens (ex. Neo4j, Amaazon Neptune).

**Data Distribution**

- Partitioning data, or sharding, requires an algorithm to distribute data across multiple nodes.
    - *Hash key*: The partition is chosen as a result of applying a hash function (modulus or consistent hashing) to the shard key and the hash is mapped to a partition.
    - *Value-based*: The partition is chosen based on the value of the shard key (ex. partitioning data based on customer country of residence).
    - *Range-based*: The shard key resides within a specific range (ex. using zip code ranges for each partition).

**Replication**

- *Leader-follower*: One replica is the leader and always holds the latest values. All writes go to the leader and is propagated to the rest of the replicas. The followers are read-only replicas.
- *Leaderless*: Any replica can handle both reads and writes. When a replica receives a write, it becomes the request coordinator that update adn is responsible for ensuring the other replicas get correctly updated.
- Replica consistency is often a major issue in distributed systems when ensuring that updates are propagated to replicas accurately especially when varying latencies and network or hardware failures can occur.
- *Strong consistency*: When a database ensures all replicas always have the same value and the client must wait until all replicas are modified before an update is considered successful.
- *Eventual consistency*: A client may only wait for one replica to be updated and trust the database to update the rest. However, there is a window of time when the replicas are inconsistent.

**CAP Theorem**

- If the network is operating correctly, a system can be both consistent and available.
- If a network partition occurs (the network drops or delays messages sent between nodes in a database), a system can either be consistent (CP) or available (AP).
    - The system may be consistent but not be available since an error is returned due all replicas not having the same values.
    - The system may be available but not consistent since the application still functions but may see different data based on the node accessed.

## Eventual Consistency

- *Read your own writes (RYOWs)*: a property of a system that ensures that if a client makes a change to the data, the updated value is guaranteed to be returned by any subsequent reads from the same client.
- *Tunable consistency*: consistency can be configured in that you can set the request coordinator to wait until *W* out of *N* repicas are updated before returning success to the client. *R* is the number of replicas to read from before returning a value.
- *Quorum reads and writes*: simply means that a majority of replicas will be queried whenever a read or write occurs. For example, with three replicas, a write must succeed at two replicas to be considered successful, and a read must access two replicas to determine the most up to date value.
    - A quorum fails if one does not exist but for database systems that favor availability over consistency, *sloppy quorums* are supported in which a failed quorum write can be temporarily stored at another database node until it can update the proper replicas. Implemented by DynamoDB, Cassandra, Riak, and Voldemort.

## Strong Consistency

- A class of distributed databases that provide strongly consistent systems, also known as NewSQL or distributed SQL, ensure all clients see the same, consistent value once it has been updated which eliminates many of the complexities inherent in eventually consistent systems.

**ACID**

- *Atomicity*: All changes to the database must be executed as if they are a single operation (*commit* or else *roll back*).
- *Consistency*: Transactions will leave the database in a consistent state. For example, if an online purchase succeeds, the number of items in stock for that product is decreased by the number purchased.
- *Isolation*: Transactions are isolated from each other by the database acquiring and releasing locks on the data objects being accessed before another transaction can access them.
- *Durability*: If a transaction commits, the changes are permanent and stored on disk.

**VoltDB**

- Built on a shared-nothing architecture where relational tables are sharded using a partition key and replicated across nodes.
- Low latencies are achieved by maintaining tables in memory and asynchronously writing snapshots of the data to disk. As a consequence, this limits the database size to the total memory available in the cluster of VoltDB nodes.
- Each table partition is associated with a single CPU core and all requests to this partition occurs in a strict single-threaded manner to alleviate contention concerns and the overheads of locking.

**Google Cloud Spanner**

- Designed to be a strongly consistent, globally distributed SQL database, which Google refers as *external consistency*.
- From the programmer's point of view, Spanner behaves indistinguishably from a single machine database.
- Spanner partitions database tables into splits (shards) and one machine can host multiple splits. Splits are also replicated across multiple availability zones to provide fault tolerance and are kept consistent using the Paxos consensus algorithm.
- Data is dynamically re-partitioned across machines as data volumes grow or shrink and data is migrated to new locations to balance load.
- An API layer processes user requests as an optimized, fault tolerant lookup service to find the machines that host the key ranges a query accesses.
- Spanner implements the TrueTime service which equips Google data centers with satellite connected GPS and atomic clocks to provide closely synchronized times with a known upper bound clock skew (reported around 7 ms). All data objects in Spanner are associated with a TrueTime timestamp that represents the commit time of the transaction that last mutated the object to help ensure the order of updates.
- Open source implementations of Google Cloud Spanner include CockroachDB and YugabyteDB but has lower consistency guarantees due to not requiring custom TrueTime-style hardware.

**Calvin Database System**

- A unique approach for a strongly consistent database where transactions are preprocessed and sequenced so that they are executed by replicas in the same order. This is known as deterministic transaction execution.

## Distributed Databse Implementations

**Redis**

- Main attraction is its ability to act as both a distributed cache and data store.
- Redis maintains an in-memory data store, known as a *data structure store* and is essentially a disk-backed cache with trailing persistence.
- To provide data safety, Redis can be configured to periodically dump the memory contents to disk or to log every command to an append-only file (AOF) and is persisted every second. Using both approaches provides the greated data safety guarantees.
- The core Redis structures are *Strings*, *Linked lists*, *Sets and sorted sets*, and *Hashes*.
- Redis is designed for low latency responses and high throughput as the primary data is stored in memory, making for fast data access.
- Redis trades off data safety for performance. It can be configured for improved data safety but at the expense of performance under heavy write loads. Redis probably shouldn't be used as a primary data store if data loss is not an option.
- Redis Cluster is the primary scalability mechanism for Redis and allows up to 1,000 nodes to host sharded databases distributed across 16,384 hash slots.
- Redis replication provides eventual consistency by default based on asynchronous replication.

**MongoDB**

- The initial popularity of MongoDB was driven by its ease of programming and use.
- MongoDB documents are basically JSON objects with a set of extended types defined in the Binary JSON (BSON) specification.
- To scale horizontally, you can choose between hash-based and range-based sharding. You define a shard key for each document based on one or more field values then MongoDB choosed a database shard to store the document in based on either a hash function applied to the shard key or based on a shard key range within which the key resides.

**Amazon DynamoDB**

- A fully managed database in that replicated database partitions are automatically managed and data is repartitioned to satisfy size and performance requirements.
- Individual data items comprise nested, key-value pairs, and are replicated three times for data safety.
- The point-in-time recovery feature automatically performs incremental backups and stores them for a rolling 35-day period.
- Full backups can be run at any time with minimal effect on production systems.
- You are charged both for storage and for every read and write made to the database.
- In general, the more you ask DynamoDB to do for you, the more you pay.

**References**

- *Principles of Distributed Database Systems*, 4th Edition (Springer, 2020) by M. Tamer Ozsu and Patrick Valduriez.









# How Netflix uses Java  - 2025


## Architecture

- Netflix streaming
 - High RPS (requests-per-second)
 - Multi Region -> 4 AWS regions
 - Large fanout to backend services
 - Retry on failure - aggresive timeouts to reduce latencies, but with retries; some failure are ok - pass without noticing
 - Non-relational data stores - multiregion + data is not strucutured in a relational manner


 - Internal enterprise/studio apps (for movie studio, movie production)
    - Low RPS
    - Single region
    - Relational data stores
    - Failure not acceptable



- Netflix streaming
1. UI fires a GraphQL query (via HTTP) to the API gateway (Zuul)
2. API GW passes it (via HTTP) to GraphQL federated gateway (behind the scene, that query involves multiple GraphQL schemas)
3. Services can register themselves to the federated GraphQL gateway. Then it knows where to send a query when some federated query is received. That means that a single federated query can hit multiple services behind the scene based on the entity ownership. All these services are called DGS - Domain Graph Service; Spring Boot service that uses DGS framework to talk over GraphQL
4. DGS services usually fire a fanout gRPC calls to other backend services. Devices (UI) use HTTP as a backbone comm protocol, so that's why the use GraphQL and on the other hand Java-2-Java service communication is based on gRPC due to its efficiency (binary protocol)
- many data stores - EVCache, Kafka, Cassandra


- Netflix Studio/Enterprise
1. UI fires a GraphQL query (via HTTP) to the API gateway (Zuul)
2. API GW passes it (via HTTP) to GraphQL federated gateway (behind the scene, that query involves multiple GraphQL schemas)
3. GraphQL federated gateway passes it to DGS services
4. these DGS services use relational database (Postgres)


- Netflix uses Java for other things like:
1. Open Connect
2. Encoding pipelines
3. Stream processing (Spark etc)
4. Data stores

The Netflix Open Connect program provides opportunities for ISP partners to improve their customers' Netflix user experience by localizing Netflix traffic and minimizing the delivery of traffic that is served over a transit provider.

There are two main components of the program, which are architected in partnership with ISPs to provide maximum benefit in each individual situation: embedded Open Connect Appliances and settlement-free interconnection (SFI).


## Java @Netflix
- JDK 8 to JDK 17 -> big migration
    - migrate all services to Spring Boot (3000 applications); they used some older, not that popular framework
    - Patch unmainted libraries for JDK compatibility
- Most high RPS services are on JDK 21 or 23 (so they can use new garbage collectors)

### JDK 17 performance gains
- G1 garbage collector is better compared to JDK 8
- About 20% less CPU spent on GC

### JDK 23 performance gains
- now using Generation ZGC as default GC
- more predictable GC, allowing to run more "hot"; it doesn't do any stop-the-world garbage collection events
- non-generational GC scan the complete heap (slow)
- by their benchmark, with G1 their stop-the-world garabge collections took about a second, second and a half. In that period, the service just rejects the traffic -  a lot of timeouts with their aggressive timeout policies
- when these stop-the-world garbage collections don't happen, timeouts don't happen either, or better said, they are much less frequent


### JDK 21 Virtual Threads
- they first added the virtual thread support in their Spring Boot framework and in DGS framework to make them running under the hood, so the way the code is written is not changed at all
- lightweight parallelizable units of work should be assigned to virtual threads and not to actual threads for the sake of the proper workload management - overkill when we use an actual thread to handle a lightweight unit of work


### Virtual threads + Structured concurrency will replace Reactive
- RxJava is developed in Netflix in most of its part
- they are completely asynchronous, used it all over the place - cons of Reactive, the code is more complex + debugging too; they backed out of using Reactive, reduced its usage  


Problem: Thread pinning and possible deadlocks
    - Mixing synchronized and ReentrantLocks can lead to deadlocks due to thread pinning
    - Some Virtual Threads are pinned to a Platform Thread while waiting on a lock, but no more Platform Threads are available to run the Virtual Thread on that owns the lock - all Platform threads are used by Virtual threads (these are pinned because of the synchronized keyword). However all these threads would wait on a lock, but that lock is owned by another Virtual thread which cannot be executed because there is no Platform thread for it to be run (all are pinned by Virtual threads) -> deadlock.
    - this issue is fixed


### Spring Boot Netflix
- Open Source Spring Boot + Netflix modules for further features and integration

### Deployment
- AWS or Titus (in-house Kubernetes)
- "exploded" JAR with an embedded Tomcat
- Not using native images
- Experimenting with AOT and Leyden

- they are not using Webflux (no to reactive)
- Javax to Jakarta - Javax works with Spring Boot 2, but not with Spring Boot 3 (based on Jakarta libs)
- this is fine if you are dealing with the service; but is bothersome if you deal with the library (which should work with both Spring Boot 2 and Spring Boot 3)
- They use Gradle transform to change the javax -> jakarta when used with Spring Boot 3, practically it changes the byte code
- Javax and Jakarta are 1:1 mapped, only the package name is different, classes are the same

- They open-source DGS in 2020
- They do not use REST

- GraphQL
    - flexible schema to query data
    - think in "data", not in "methods"
- gRPC
    - highly performant server-to-server calls
    - think "methods" not "data"

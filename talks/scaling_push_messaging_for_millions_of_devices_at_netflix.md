# Scaling Push Messaging for Millions of Devices @Netflix

125 millions of devices - personalized movie recommendations
How they push those recommendation updates - their algoritms run constantly, computing fresh lists

Earlier, they used the poll mechanism so the UI polled the server. How often to do that - data freshness vs eficiency (overloading servers)

Now they use push mechanism - the server initiates the connection and pushes messages to clients (they increased the efficiency for 12% in this way)
They do it in the background.
Zuul Push - open-source, based on web sockets and server side events

Architecture:

1. Zuul push servers (sit on the edge network)
2. SSE/Websocket connections are maintainted with clients for the entirety of their lifetime
3. Push registry - keeps the map of server-to-client connections, i.e. which server serves messages to which client
   Just two things to store - device ID, client ID... and the internal 10/24 address of a push server.
   Push lib - the library used to push messages
   Push message queue - stores these messages
   Message processor - delivers a message, hires a push server based on the registry which then delivers a message to a particular client; if the client is not present in the registry, that means it is not connected to any server and it rejects the message
   [1]
   Zuul push server - maintains about 10 milion of concurrent connections at a single time (at peak, 6 years ago); async I/O approach based on Zuul cloud gateway
   C10k challenge - the server should handle 10,000 concurrent open connections

Thread per connection approach won't work - if we have 10k threads, that will exhaust the memory and CPU on constant context switching;
[](images/thread_per_connection_vs_async_io.png)

Async I/O approach is much more efficient -> single thread used - we register the callback which gets called when read or write operation is executed, i.e. when the socket is actually used to transmit data
They used Netty at that time (switched to vritual threads in the meantime - Cassandra and Hadoop use Netty as well. The code is much more complicated now compared to thread-per-connection approach.
The client when it connects to Zuul, it has to autenticate and identify itself

[2] Push registry
E.g. they use Redis for such purpose
Characteristics of pthe push registry:
a) low read latency
b) record expiry; TTL - they do not want to keep phantom records (saying that a client is connected to some server, when it's not)
c) sharding
d) replication

Redis/Cassandra/Amazon DynamoDB

They use Dynomite -> based on Redis, supports autosharding, cross-region replication

[3] Message queue

- They use Kafka
- Fire and forget -> push library pushes a message and that's it; they don't care about the delivery

[4] Cross-region replication
Kafka replicates message in 3 regions

If they use a single queue, there could be a priority inversion - i.e. the message of high priority is in pending state because there is a bunch of low-priority messages in front of it
They use the priority queue

- Apache Mesos for container management
- multiple instances of the message processor

[5] Operating Zuul push

- long lived stable connections -> Zuul is therefor stateful

  - great for the client efficiency
  - terrible for quick deploy/rollback

- if we deploy new Zuul cluster, nothing will happen automatically, they have to migrate all existing clients to it, because they connect once at a startup time and the same connection hangs and it's used when needed
- when all clients connect to a new cluster, we get the thundering herd problem
- they changed the strategy, avoid sticky connections
- that's why the server tear down connections periodically, so clients have to reconnect from time to time
- to prevent the recurring thundering herd, instead of having the static constant connection break time, they randomize it within certain boundaries

- extra credit: ask client to close its connection; to save the file descriptors - on Linux, if the server closes the TCP connection, the file descriptor will be occupied for another 2 minutes which is bad for server who has thousands of connections. That's why they ask the client to close the connection. If it doesn't comply within a given time frame, the server forcefully do that.

How to optimize push server:

1. most connection are idle, most of the time
   they fine tuned the TCP connection parameters, JVM parameters... -> thundering herd crashed their big AWS server

2. Goldilock strategy: m4 large AWS instance is great for them - 8gb of RAM and 2 vCPUs -> 84k connections at a single time

- optimize for the total cost, not for the instance count; more number of cheaper servers is better than less stronger servers

- How to auto-scale?
  - RPS or CPU? - NO
  - the number of open connections (avg num of connections)
  - you export the parameter in Cloud Watch and you can scale based on it
- Amazon ELB cannot proxy web sockets; it treats is any HTTP request, as soon as it returns the response, it closes the connection
- they changed to TCP load balancer (the default config of ELB is HTTP LB); they neglect headers, methods, paths...
- in the meantime Amazon published ALB which support web socket load balancing

1. Recycle connections after tens of minutes
2. Randomize connection's lifetime
3. More number of smaller servers is better than a few big servers
4. Auto-scale on number of open connections per box
5. Websocket aware vs TCP load balancer

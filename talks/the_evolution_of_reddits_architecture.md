# The Evolution of Reddit's Architecture

- initially they had r2 - the oldest component of Reddit, started in 2008, written in Python
- frontend -> Node.js frontend applications
- New backend services:
  - Written in Python
  - Splitting off from r2
  - Common library/framework to standardize
  - Thrift or HTTP depending on clients
  - Services: API, Search, Thing, Listing, Rec.
- CDN - fastly
- r2 deep dive:

  - Load balancers - HAProxy
  - apps
  - jobs (RabbitMQ)
  - cache (Redis and Memcache) - Thing is model component backed by Postgres with cache sitting (memcached) in front of it
  - Cassandra (new features)

- Listings

  - they run the query and cache results (post ids), then they retrieve the actual links from database by primary keys (ids)
  - cached listings are invalidate when you change something (you submit something or you simply vote)
    - invalidates a lot of cached queries
    - they do an expensive anti-cheat processing
    - deferred to offline job queues with many processors
  - instead of rerunning the query (which is slow) they mutate the result in place
    - they do not sore only votes, but also other info (sort info - number of votes), so whenever vote happens, they update the entry in place and re-sort it
    - that requires locking (needed because they write data back to cache)

- Cache

  - this isn't really cache anymore, it is a denormalized index of links
  - data is persisted to Cassandra, reads are still served from memcached

- Votes are processed asynchronously (i.e. stored in queues)

  - at peak traffic hours, these vote queues are piled up
  - a vote can wait in queue for hours before being processed (was noticeable by users)
  - the problem was with those locks upon the cached query mutation
  - solution: queue partitioning
    - they partitioned queues into multiple queues, so votes are put into different queues based on the subreddit of the link being voted on (ID mod num of queues)
  - another problem which made troubles is when the vote is not for the subreddit, but the domain of a submitted link (very populare domain was being submitted in many subreddits); e.g. domain: imgur.com
    -> solution: they split up processing, so queries are divided into subreddit queries, domain queries and profile queries and then they are routed to different queues
  - locks are bad for throughput, if you have to use them, use partitioning

  - Thing:

    - PostgreSQL cluster (primary and secondaries - async replication)
    - r2 connects to dbs directly, prefers to use replicas for read ops for better scale
    - if a query filed, it assumes that the node is down and it tries not to read from it again, i.e. removes it from the connection pool

    - whole thing object is serialized and added to memcached; on reads, it firs reads data from cache and hits Postgres only on cache miss
    - r2 writtes changes directly to memcached at the same time does to Postgres

  - Incident 2011

    - one of the replicas crashed
    - they removed it from the pool and rebuilt it
    - the problem that got born is - cached values were pointing to some ids that didn't exist
    - primary was saturating its disk, IOps was affected - they upgraded hardware of the primary node

  - The failover code:

  ```
  live_databases = [db for db in databases if db.alive]
  primary = live_databases[0]
  secondaries = live_databases[1:]

  ....
  if query.type == "select":
      random.choice(secondaries).execute(query)
  elif query.type in ("insert", "update"):
      primary.execute(query)
  ```

  - if a primary was not alive, it would write to a secondary - BUG
    - they wrote data to secondary; permissions were not set properly...
    - they stored it in cache
    - it got affected, they rebuilt it, the data is gone (still present in cache)
    - proper permissions as a protection mechanism

- Comment trees:
  - if comment thread gets deeply nested, they are offloading it to an offline job worker called fastlane
  - massive comment threads can make comments across the site slow
  - an incident - a huge thread blocked the whole site -> actions like voting, commenting, posting links were frozen (message queues were completely memory-exhausted)
  - switching to fastlane allowed new messages to skip the queue; tree is now inconsistent (this caused recompute messages to flood the queue on every pageview - messages are without a parrent, asking the BE to recompute their parrent)
  - they restarted RabbitMQ, lost messages and started over
  - they now set the maximum queue length, so that no queue can consume all resources -> user-visible, but scope of impact limited; quotas are important for isolation
- Autoscaling
  - they did the autoscaling by user the autoscaling groups on AWS, but every node had a deamon that registered the node presence in ZK
  - also memcached servers registered themselves in ZK, so if a node failed, the automatic replacement happens

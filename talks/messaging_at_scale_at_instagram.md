# Messaging at Scale at Instagram

- fanout write, when media is published, read all followers and write the media ID to Redis (key is an account id)
- O(1) read cost, O(n) write cost (n followers)
- reads outnumbers writes 100:1 or more
- what if someone has so many followers
- they used async workers - broker distributes write tasks to workers (resilient and scalable)
- chained tasks - 10,000 followers per task, tasks yield successive tasks; fine grained load balancing
- They used Gearman - a simple, purpose-built task queue - it was slow, inefficient, single core, didn't scale well, memory crashes etc.
- they switched to Celery - distributed task framework, highly extensible, pluggable, rich support, works well with Django...
- Redis was not an option
  - very fast and efficient (they were already using it)
  - polling for task distribution
  - messy non-synchronous replication
  - memory limits task capacity (runs in-memory)
- Beanstalk was not an option either
  - purpose-built task queue
  - very fast, efficient
  - pushes to consumers
  - spills to disk
  - no replication
  - useless for anything else
- RabbitMQ
  - reasonably fast, efficient
  - spill to disk
  - low-maintenance synchronous replication
  - excellent celery compatibility
  - supports other use cases
  - they didn't know Erlang
- Their RabbitMQ setup
  - clusters of two broker nodes, mirrored
  - monitoring: Sensu
  - graphing: graphite & statsd
- Metrics: ~4000 tasks/s, ~25000 app threads publishing tasks

- Celery + concurrency models
  - multiprocessing (pre-fork) ✔️
  - eventlet ❌
  - gevent ✔️
  - threads ❌
- `celeryd_multi` - run multiple workers with different parameters (such as concurrency settings)

- `gevent` - they were using it for network bound (Facebook API, Tumblr API, Various Background S3 tasks, checking URLs for spams)

- Problems:
  - Slow tasks monopolizes workers -> they separated slow and fast tasks
  - Tasks fail sometimes -> wait 60 seconds before retrying
  - Worker crashes can lose tasks:
    - Flow:
      1. worker starts the task
      2. they send the ack to broker
      3. broker forgets the task
      4. worker finishes task
         Between 3 and 4 if the worker crashes, the task is gone, it won't be repeated
         Solution:
    1. worker starts the task
    2. worker finishes task
    3. they send ack
- Tasks should be idempotent
  "...it is impossible for one process to tell whether another has died (stopped entirely) or is just running very slow"
- Impossibility of Distributed Consensus with One Fault Process (Fischer, Lynch, Patterson 1985)

- Retry -> duplicates
- Do not retry -> maybe the task will not be processed at all

- Overloaded brokers were dropping tasks
  - Publisher Confirms makes broker send acks back on publishers
  - can cause duplicate tasks

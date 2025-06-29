# Caching at Netflix - The Hidden Microservice

- caching is multilayered

### EVCache

- Ephemeral Volatile Cache
- KV store optimized for AWS and tuned for Netflix use cases
- Distributed, sharded, replicated key-value store
- Based on Memcached
- Tunable in-region and global replication
- Resilient to failure
- Topology aware
- Linearly scalable
- Seamless deployments

### Why optimize for AWS?

- instances disappear
- zones fail
- regions become unstable
- network is lossy
- customer requests bounce between regions
- failure happen, they test all the time

### EVCache use @Netflix

- Hundreds of terabytes of data
- Trillions of ops/day
- Tens of billions of items stored
- Tens of millions of ops/sec
- Millions of replications/sec
- Thousands of servers
- Hundreds of instances per cluster
- Hundreds of microservice clients
- Tens of distinct clusters
- 3 regions
- 4 engineers

## Architecutre

![](caching_arch.png)

- Prana (sidecar) written in Java - meant for service that are not written in Java, so they can be attached to Nutanix metrics
  ![](images/caching_mesh.png)

#### Read

- When reading, it tries the closest one and after that it falls back to other instances
  ![](images/cache_read.png)

### Write

- Whenw writing, it writes to all three replicas
  ![](images/cache_write.png)

### Use case: Lookaside cache

- relatively slow service, relatively fast cache -> tries first in the cache, then the service
- if the value is not found in the cache, goes to the service, it reads it from the database and with returning the result, it updates the cache -> before returning the response or asynchronously
  ![](images/lookaside_cache.png)

### Transient data store

- when we don't have any data store apart from cache, e.g. we use it to update the session
  ![](images/cache_transient_data_store.png)

### Primary store

- an example: they run on nightly basis the homepage of every profile/every user; cache acts like buffer here
  ![](images/primary_store.png)

### High volume && high availability

![](images/hv_and_ha.png)

## Pipeline of personalization

- personal home page computation

![](images/pipeline_of_personalization.png)

## Polyglot clients

![](images/polyglot_clients.png)

## Additional features

- Global data replication
- Cache warming
- Secondary indexing
- Consistency checking
- All powered by metadata flowing through Kafka

### Cross-region replication

- Netflix has 3 operational regions
- this happens for every 2 regions
  ![](images/cross_region_replication.png)

### Cache warming

- steady state -> app uses one cache replica
  ![](images/steady_state_1.png)
- if we bring up more cache replicas, we have to warm them (populate with data)
  ![](images/cache_warming.png)
- Cache warmer does that; once the warming is complete, Cache warmer is shut down
  ![](images/steady_state_2.png)

### Moneta

- goddess of memory
- Juno Moneta: the protectress of Funds for Juno

- Evolution of the EVCache server
- EVCache on SSD
- Cost optimizatio
- Ongoing lower EVCache cost per stream

#### Old server

- stock Memcached and Prana (Netflix sidecar)
- All data stored in RAM in Memcached
- Expensive with global expansion/N + 1 architecture (if one region goes down, they can still serve the traffic equally well)

![](images/old_server.png)

##### Optimization

- global data means many copies (3 copies x 3 regions = 9 copies)
- access patterns are heavily region-oriented
- in one region:

  - hot data is used often
  - cold data is almost never touched

- keep hot data in RAM, cold data on SSD
- Size RAM for working set, SSD for overall dataset

#### New server

- adds Rend and Mneonic
- Still looks like Memcached
- Unlocks the cost-efficient storage & server-side intelligence

##### Rend

- high-performance Memcached proxy & server
- written in Go
  - powerful concurrency primitives
  - productive and fast
- manages the L1/L2 relationship
  - L1 - Memcached
  - L2 - Mnemonic
- tens of thousands of connections

![](images/rend.png)

#### Parallel locking

- many parallel requests, but no two concurrent modifications for the same key

#### Mnemonic

- manages data storage on SSD
- Resuses Rend server libraries
- Maps Memcached ops to RocksDB ops

- Consists of:

1. Rend server core lib (Go)
2. Mnemonic Op Handler (Go)
3. Mnemonic Core (C++)
4. RocksDB (C++)

#### Why RocksDB?

- fast at medium to high write load
  goal was 99% read latency below 20 ms
- Log-Structured Merge minimizes random writes to SSD
  - writes are buffered in memory
- Immutable Static Sorted Tables

- first writes to buffer, then asynchronously flushes to disk (in static files) <br>
  ![](images/rocksdb.png)

#### How they use it?

- FIFO compaction (more recent files are stacked on top and older ones are deleted)
  - more suitable for their precompute use cases
- Bloom filters and indices pinned in memory
  - Trade L1 space for faster L2
- Records sharded across many RocksDB per instance
  - reduces number of files checked, decreasing latencies

![](images/rocksdb_netflix.png)

#### FIFO limitation

- FIFO compaction not suitable for all use cases
  very fequently updated records may push out valid records
- Future: custom compaction or level compaction

![](images/compaction.png)

![](images/moneta_in_production.png)

![](images/latencies.png)

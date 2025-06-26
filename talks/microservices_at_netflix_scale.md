# Microservices at Netflix Scale

- all benefits, no costs?
- 120+M hours of stream on daily basis, more than 4 billion on monthly basis, 1/3 of stream traffic in North America, over 500 microservices
- about 1200 engineers
- started in 2008
- necessity to move to cloud started their microservices journey

- Initially, monolith in war format; Oracle DBMS
- many problems; database failure happened in 2008, 4 days to recover

Buy vs build

- use or contribute to OSS technologies first
- only build what you have to...

Services should be stateless (except the persistence/caching layers)

- must on rely on sticky sessions
- prove by chaos testing

Scale out vs scale uo

- scaling up has a limit
- horizontal gives you a longer runway

Redundancy and isolation for resiliency

- make more than one of anything
- isolate the blast radius for any given failure

Automate destructive testing

- Simian army
- Started with Chaos Monkey

- kill one instance and retry, if the effect is the same, it's really stateless -> Chaos monkey can help with that
  -> to verify stateless

Data - from RDBMS to Cassandra

- NoSQL at scale
- open-source -> multi-regional and multi-directional (Netflix contribution)

- available
- partition tolerance
- tunable consistency -> local quorum to global quorum

- async replication between replicas

Microservices - benefits

- Netflix priorities:

  1.  Innovation -> high velocity
  2.  Reliability
  3.  Efficiency

- Innovation: tight coupling between teams doesn't work
- The goal is end-to-end ownership; every team has the same ownership cycle

- Microservices is an org change (that's the hard thing)
- Migration doesn't happen overnight
  - living the hybrdi world
  - supporting 2 tech stacks - double the maint - multi-master data replication

IPC is cruical for loose coupling - inter-process communication

- common language between the services
- establishes the contract of interaction
- Spinaker - automates the process of deployment

Caching to protect DBs

1. Read from cache
2. On cache miss call service
3. Service calls DB and responds
4. Service updates the cache

Operational visibility matters

- if you can't see it, you can't improve it
  observe -> orient -> decide -> act -> observe
- they automated telemtry checking, too many information to process manually

Reliability matters

- we strive for 4 9s of availability
- that leaves only 52 minutes of downtime per YEAR

Cascading failures

- 99% of availability -> 500 services in cascade
- 99^500 = 0.00657
- circuit breaker to break the failing chain
- Hystrix + Chaos monkey for testing
- they trigger failures on whole regions (randomly select the whole region and bring it down) in production

Microservices - resources:

- Netflix OSS

# How Shopify’s engineering improved database writes by 50% with ULID

## Idempotency

- idempotent request - one that doesn't change the state of an app/backend, e.g. GET request which by definition must be idempotent (on the other hand, POST is not idempotent)
- make your backend idempotent - e.g. implement `upsert` - creates a record if it doesn't exist, otherwise, it updates it

> An idempotency key needs to be unique for the time we want the request to be retryable, typically 24 hours or less. We prefer using an Universally Unique Lexicographically Sortable Identifier (ULID) for these idempotency keys instead of a random version 4 UUID. ULIDs contain a 48-bit timestamp followed by 80 bits of random data. The timestamp allows ULIDs to be sorted, which works much better with the b-tree data structure databases use for indexing. In one high-throughput system at Shopify we’ve seen a 50 percent decrease in INSERT statement duration by switching from UUIDv4 to ULID for idempotency keys.

- so, when we retry the same request, the same request key goes with it, making it idempotent (like when we resend the same form data)
- databases like ordered things, so sequences are good. Generating an incremental id (sequence) is expensive, because you have to talk to database to give you the current sequence value

## UUID vs ULID

- UUIDs are unique - a great thing, especially in distributed systems
- ULID = 48 bits (6 bytes) of timestamp make ULID sortable

## Clustered Index

- they use MySQL
- MySQL primary keys are called `clustered index` - e.g. if we have integers for primary keys, then in B+ tree leaf nodes we have pages with rows themselves, practically data is stored in those leaf nodes (sorted rows by PK); e.g. if you are searching the row with id 7, you will find the complete row in the leaf node of a clustered index - you find that row 7 and all rows that are next to it by PK, e.g. 5,6,8, 10, 12.... which is great for range scans -> we have a index-only scan

## Why UUID4 inserts are slow?

- what if index key is not ordered, you insert 3, 1000, 100, 20 and so on? Well, the database has to sort them, because indexes are sorted. UUID is not sorted, so its insertion is slower due to its random nature - writing UUIDs means pointing to random pages and you have to find where does that UUID live (in which page) based on that random order
- random value insertion -> random I/Os; we are writing a UUID, MySQL has to fetch a page where it should live, put it in memory (buffer pool) to execute an operation, some other UUID is written (or read) which does not live in that page, so MySQL cannot use any caching (cannot use the page it loaded before) which imposes much more I/Os = slower insertions -> we are loading pages that are not used frequently, so we tend to read the whole table with this strategy
- writes are worse than reads with this strategy
  - read a page in memory
  - write a value
  - add an entry to WAL
  - flush the page
- when the buffer pool is filled in (which in Shopify's case is easy with millions of requests), we have to flush dirty pages to disk to empty the pool = flishing is an expensive operaiton

## How ULID helps Shopify?

- with ULIDs which are sorted, once you fetch the page to insert a ULID, the next ULID will be in the same page until it is full, so the number of fetched pages is significantly smaller than when we use UUIDs; we will always append to tail of an index file
- reads are also faster, because pages are probably alread in memory; dirty page is fine (page not flushed to disk, but updated in WAL)

## Problem with tail pages

- a page is a structure in memory used to interpret db data
- in multithreading system (database is one), multiple threads are writing to the same page, so we have to be careful to do it properly - using mutexes (in MySQL they should be calling them latches); all threads are then adding to the tail when ULID is used

## Does ULID helps in all cases?

- write performance is better with ULIDs
- 128 bits (16 bytes) is a solid size
- secondary indexes point to primary index key (primary key), so that can bloat up
- indexes are loaded in memory, so bigger indexes mean more memory is taken or better said, less things can fit into memory which is of constant size (RAM)

To read: https://shopify.engineering/building-resilient-payment-systems

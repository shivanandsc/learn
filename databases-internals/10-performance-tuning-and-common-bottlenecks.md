# Database Internals: Performance Tuning and Common Bottlenecks

This module connects internal mechanisms to real performance issues and what to look at when tuning.

## Performance is usually one of these
- CPU bound (poor plans, heavy expressions, contention)
- I/O bound (cache misses, random reads, flush storms)
- lock/latency bound (contention, deadlocks, long transactions)
- memory bound (spill to disk, compaction pressure)

## Observability primitives
Most DBs expose:
- query plans (`EXPLAIN`)
- buffer/cache hit rates
- lock waits
- WAL throughput and fsync latency
- checkpoint duration and dirty page ratio
- background work (vacuum/compaction)

## Common bottlenecks and internal causes

### 1) Slow queries due to bad plans
Internal cause:
- wrong cardinality estimates
- missing/ineffective indexes
- parameter-sensitive plans

Levers:
- update statistics
- add/change indexes
- rewrite queries to be sargable
- use plan hints (if available) cautiously

### 2) Random I/O and cache misses
Internal cause:
- working set larger than buffer pool
- inefficient access patterns (N+1 lookups)

Levers:
- increase memory/buffer pool
- add covering indexes
- batch access patterns

### 3) Write stalls from checkpoints/flush
Internal cause:
- too many dirty pages
- WAL pressure
- fsync latency spikes

Levers:
- tune checkpoint intervals/targets
- faster storage
- spread writes (avoid bursts)

### 4) Lock contention
Internal cause:
- hot rows/keys
- long-running transactions holding locks
- high isolation modes

Levers:
- shorten transactions
- reduce hotspotting (shard/partition keys)
- adjust isolation/locking strategy

### 5) MVCC bloat / vacuum debt
Internal cause:
- old snapshots prevent cleanup
- long-running readers

Levers:
- avoid long transactions
- tune vacuum/GC

### 6) LSM compaction pressure (for LSM engines)
Internal cause:
- sustained high write rate triggers compaction backlog

Levers:
- tune compaction
- provision more I/O
- change workload patterns

## Practical mental model
A useful way to reason:
- Reads want: **cache hits + sequential I/O**
- Writes want: **sequential logging + smooth background flush/compaction**
- Concurrency wants: **short transactions + minimal contention**

## Recap
- Most issues map to query planning, I/O/caching, logging/checkpoints, or contention.
- Use explain plans and internal metrics to find the real subsystem causing pain.

## Common misconceptions / pitfalls
- **“Add an index and it’s fixed.”** Indexes can hurt writes and may not be used.
- **“The DB is slow because it’s using disk.”** Disk usage is normal; the question is whether it’s random I/O and why.

# Database Internals: Buffer Pool (Caching) and I/O

This module explains how databases minimize expensive disk I/O using a buffer pool, and how reads/writes flow through it.

## The I/O problem
Storage is slow compared to CPU/RAM. If every query hit disk, performance would collapse.

Databases therefore:
- cache pages in memory
- batch writes
- do sequential writes when possible (WAL, LSM)

## Buffer pool basics
The **buffer pool** (aka buffer cache) stores recently used database pages in RAM.

### What’s in a buffer frame
A buffer frame (in-memory copy of a page) usually tracks:
- page identifier (file + page number)
- pin/refcount (in use?)
- dirty flag (modified?)
- page LSN
- latch/mutex for thread safety

### Page table / mapping
A hash table maps (relation, page_id) → buffer frame.

## Read path
A typical page read:
1. Executor requests a page.
2. Buffer manager checks if it’s already cached.
   - **Hit**: return pointer quickly.
   - **Miss**: choose a victim frame, read from disk into it.
3. Latch/pin page while it’s being used.

Important: DBs typically read whole pages even if only one row is needed.

## Write path (in-place update design)
For B+tree/heap-style engines:
1. Transaction modifies a cached page.
2. Page becomes **dirty**.
3. WAL records are appended (and flushed on commit depending on policy).
4. A background writer flushes dirty pages later.

This decoupling improves latency but requires WAL + recovery.

## Eviction policies
When buffer pool is full, it must evict pages.

Common strategies:
- LRU variants (clock, segmented LRU)
- prioritize clean pages to avoid forced writes

DBs also avoid evicting pages currently pinned.

## Flushing, checkpoints, and write ordering
### Flushing dirty pages
Dirty pages are written back to disk:
- periodically
- under memory pressure
- during checkpoints

### Checkpoints
A **checkpoint** creates a recovery point by ensuring:
- enough dirty pages up to a certain LSN are flushed
- WAL prior to that point may become less necessary for recovery (implementation-dependent)

### Write ordering (WAL rule)
The **write-ahead** rule:
- WAL describing a change must be durable **before** the data page reflecting that change is flushed.

## Read-ahead and prefetch
For scans, DBs often:
- prefetch sequential pages
- use larger I/O requests
- employ scan-resistant buffer strategies (avoid polluting cache)

## OS cache vs DB cache
Some DBs rely heavily on the OS page cache; others manage their own buffer pool.

Tradeoff:
- DB-managed cache can coordinate with locking, WAL, and eviction policies.
- OS cache is simpler but less DB-aware.

## Recap
- The buffer pool is central to performance.
- Reads aim for cache hits; writes are staged in memory and persisted via WAL + background flush.

## Common misconceptions / pitfalls
- **“Bigger buffer pool always helps.”** It can, but beyond the working set it may increase checkpoint/flush costs.
- **“A flush equals a commit.”** Commits usually require WAL durability; page flush can happen later.

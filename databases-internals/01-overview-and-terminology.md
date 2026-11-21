# Database Internals: Overview and Terminology

This module builds a shared vocabulary for how most databases work internally. It focuses on *general* database engines (especially relational/SQL) and notes where NoSQL engines differ.

## What a database engine does
At a high level, a database system:

1. **Accepts requests** (SQL or API calls) over a network or local IPC.
2. **Parses and validates** those requests.
3. **Plans and executes** them efficiently.
4. **Reads/writes data** through storage structures (in-memory and on-disk).
5. **Coordinates concurrent users** safely.
6. **Recovers from crashes** without losing committed data.

## Core terms

### Logical vs physical
- **Logical layer**: schemas, tables/collections, rows/documents, indexes, SQL.
- **Physical layer**: files/pages, B-trees/LSM trees, buffer cache, WAL, compaction.

### Transaction
A **transaction** is a unit of work that the DB treats as an all-or-nothing change.

**ACID** properties:
- **Atomicity**: all changes happen or none do.
- **Consistency**: transactions move the DB between valid states (enforced by constraints).
- **Isolation**: concurrent transactions behave *as if* they ran alone (to some degree).
- **Durability**: committed data survives crashes.

### Query lifecycle (bird’s-eye)
1. **Client protocol**: connection, authentication, request framing.
2. **Parse**: produce an AST (abstract syntax tree).
3. **Rewrite**: normalize, expand views, apply rules.
4. **Optimize**: choose join order, indexes, algorithms.
5. **Execute**: run the physical plan operators.
6. **Return results**: stream rows/pages back to client.

### Page (or block)
Most disk-oriented DBs store data in fixed-size **pages** (commonly 4–16 KB). Pages are the unit of:
- disk I/O
- caching in the buffer pool
- many locking/latching operations

### Index
An **index** is a secondary structure that accelerates lookups (at the cost of extra space and write overhead).

Common index families:
- **B-tree/B+tree**: great for point lookups and range scans.
- **Hash index**: good for equality lookups (no range ordering).
- **LSM-based index**: optimized for heavy writes; reads may touch multiple levels.

### Buffer pool / cache
A **buffer pool** caches frequently used pages in memory to avoid disk reads.

Key ideas:
- cache hit/miss
- eviction policy (LRU-like variants)
- dirty page tracking and flushing

### WAL (Write-Ahead Log)
A **WAL** records changes *before* the corresponding pages are written to their final locations. This enables crash recovery.

### Replication
Copying data to other nodes for availability and read scaling.
- **Synchronous**: commit waits for replicas (lower latency tolerance, higher safety).
- **Asynchronous**: commit returns earlier (better latency, possible data loss on failover).

### Storage engine
The component that implements on-disk layout, indexes, and recovery. Some DBs can swap storage engines (e.g., MySQL historically).

## SQL vs NoSQL internals (very rough)
- **Relational DBs** often emphasize:
  - rich query optimizer
  - joins and relational algebra
  - strict schemas and constraints
  - strong transactional guarantees
- **NoSQL systems** vary widely:
  - key-value stores may have simpler query planning
  - document stores still have indexing, caching, journaling
  - distributed systems often trade strict isolation for availability

## Recap
- A DB is a coordinated system of parsing/planning/execution + storage + concurrency + recovery.
- Most internal mechanisms exist to balance: **latency**, **throughput**, **correctness**, and **durability**.

## Common misconceptions / pitfalls
- **“The DB writes directly to disk on every commit.”** Usually commits primarily append to WAL; data pages may flush later.
- **“Indexes always make things faster.”** They speed reads but slow writes and consume memory/disk.
- **“Isolation = serial execution.”** Isolation levels vary; many systems default to weaker-than-serializable isolation.

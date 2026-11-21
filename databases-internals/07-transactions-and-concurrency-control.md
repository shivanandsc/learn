# Database Internals: Transactions and Concurrency Control

This module explains how databases allow many clients to read/write simultaneously while maintaining correctness.

## The concurrency problem
If two transactions interleave operations, you can get anomalies such as:
- **lost update**
- **dirty read**
- **non-repeatable read**
- **phantom**

Concurrency control defines rules to prevent (some or all) anomalies.

## Isolation levels (SQL mental model)
Common levels and what they prevent (simplified):
- **Read Uncommitted**: allows dirty reads (rare in practice).
- **Read Committed**: prevents dirty reads.
- **Repeatable Read**: prevents non-repeatable reads; phantoms depend on engine.
- **Serializable**: strongest; transactions behave as if serialized.

Implementations vary widely, especially around “repeatable read”.

## Two-Phase Locking (2PL)
### Idea
Use locks on data items:
- **S (shared)** lock for reading
- **X (exclusive)** lock for writing

**Two-phase** means:
- growing phase: acquire locks
- shrinking phase: release locks

Strict 2PL keeps write locks until commit, helping recoverability.

### Pros/cons
- Pros: strong correctness, straightforward.
- Cons: deadlocks, lock contention.

### Deadlocks
When transactions wait in a cycle.
DBs handle via:
- detection (wait-for graph) + abort one
- timeouts
- deadlock avoidance (wound-wait / wait-die)

## MVCC (Multi-Version Concurrency Control)
Many modern DBs use MVCC so readers don’t block writers.

### Core idea
- Writes create **new versions** of rows.
- Readers see a **snapshot** (consistent view) based on timestamps/transaction IDs.

### Visibility rules (conceptual)
A row version is visible if:
- it was created by a committed txn before the reader’s snapshot
- and not deleted/overwritten by a committed txn before the snapshot

### Where old versions live
- in the same table pages (version chains)
- in undo logs
- in separate version storage

### Garbage collection
Old versions must be cleaned up:
- vacuuming
- pruning
- undo log truncation

If GC can’t keep up, storage bloat grows.

## Latches vs locks (again, but important)
- **Latches**: short critical sections protecting in-memory structures.
- **Locks/MVCC rules**: enforce transaction isolation and logical correctness.

## Optimistic concurrency control (OCC)
Some systems:
- run transactions without heavy locking
- **validate** at commit (check read/write conflicts)
- abort and retry on conflict

Works well when conflicts are rare.

## Recap
- Isolation defines which anomalies are prevented.
- 2PL and MVCC are common strategies; many DBs combine them.

## Common misconceptions / pitfalls
- **“MVCC means no locks.”** Writers still need coordination; many systems still lock for writes and for constraints.
- **“Serializable is always too slow.”** Some engines implement efficient serializable modes (often still with tradeoffs).

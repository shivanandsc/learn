# Database Internals: Logging, Checkpoints, and Crash Recovery

This module explains how databases recover to a consistent state after crashes, focusing on WAL-based recovery.

## The durability problem
Crashes happen:
- power loss
- process crash
- OS panic

The DB must ensure:
- committed transactions remain
- uncommitted changes do not appear

## Write-Ahead Logging (WAL)
### Principle
Before a modified data page is written to disk, the log records describing the change must be persisted.

Why it helps:
- log writes are sequential (fast)
- data page writes can be delayed and batched

### WAL records
WAL may contain:
- logical changes (e.g., “insert key K”) or
- physical changes (byte-level diffs)

Each record has an LSN (log sequence number).

## Commit protocol (conceptual)
On commit:
1. Ensure all required WAL records are written.
2. Force WAL to stable storage depending on durability settings (often `fsync`).
3. Mark transaction committed (in log and/or in commit table).

Data pages may still be dirty in memory; WAL guarantees redo can restore them.

## Checkpoints
A checkpoint bounds recovery time.

Conceptually, at checkpoint time the DB:
- notes a checkpoint LSN
- flushes enough dirty pages so recovery need not redo from the beginning

Different DBs implement fuzzy checkpoints (not stopping the world).

## Crash recovery: REDO and UNDO
A common model:

### Analysis
- determine which transactions were in-flight
- reconstruct needed metadata (dirty page table, etc.)

### REDO
- reapply changes from WAL to bring pages up to date
- uses page LSN to skip already-applied updates

### UNDO
- roll back effects of transactions that didn’t commit

Some MVCC systems avoid physical UNDO at crash time by using commit markers and version visibility.

## Idempotence and correctness
Recovery must be safe if operations are applied more than once.
- Page LSN checks make redo idempotent.

## Group commit
To increase throughput:
- batch commits
- do one WAL flush for multiple transactions

Tradeoff: slightly higher commit latency for some transactions.

## Recap
- WAL converts random writes into sequential appends.
- Checkpoints reduce recovery time.
- Recovery generally does analysis → redo → undo (or MVCC-equivalent).

## Common misconceptions / pitfalls
- **“Checkpoint means all data is on disk.”** Often only enough is flushed to bound recovery.
- **“Turning off fsync is fine.”** It can lose committed transactions on crash/power loss.

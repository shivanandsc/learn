# Database Internals: High-Level Architecture and Components

This module explains the major subsystems inside a typical database server and how requests flow through them.

## Typical database server architecture
A common decomposition:

1. **Client/Protocol Layer**
2. **SQL/API Layer (Parser/Rewriter)**
3. **Optimizer (Planner)**
4. **Execution Engine**
5. **Transaction Manager**
6. **Concurrency Control (Locks/MVCC)**
7. **Storage Engine**
8. **Buffer/Cache Manager**
9. **Logging & Recovery (WAL + checkpoints)**
10. **Background workers** (flush, compaction, vacuum, replication)

Not every DB uses exactly these boundaries, but the responsibilities are similar.

## Request flow (end-to-end)

### 1) Connection & authentication
- Establishes a session/connection.
- Authn/authz checks.
- Session state is initialized (default schema, isolation level, timeouts).

### 2) Parse
- Converts SQL text into an AST.
- Checks syntax and basic semantics (e.g., column exists).

### 3) Rewrite
- Expands views.
- Applies rules (predicate pushdown, constant folding).
- Normalizes expressions to simplify planning.

### 4) Optimize (planning)
- Builds candidate physical plans (different join orders, indexes, algorithms).
- Chooses a plan using a **cost model** driven by statistics.

### 5) Execute
The executor runs operators such as:
- table/index scans
- filters
- joins
- sorts/aggregations

Execution can be:
- **tuple-at-a-time** (classic iterator model)
- **vectorized** (process batches for CPU efficiency)
- **compiled/JIT** (generate code for hot queries)

### 6) Transaction & concurrency integration
During execution, the system:
- establishes a transaction snapshot (for MVCC)
- acquires locks/latches as needed
- validates visibility rules
- writes WAL records for changes

### 7) Storage reads/writes
- Reads typically go through the buffer pool to fetch pages.
- Writes modify cached pages (becoming **dirty**) and append to WAL.

### 8) Commit/rollback
- **Commit**: ensure WAL is durable (fsync depending on durability settings), release locks, finalize MVCC metadata.
- **Rollback**: undo changes (either logically via MVCC or physically via undo logs).

## Key internal boundaries

### Transaction manager vs storage engine
- **Transaction manager**: “what changes are part of this transaction?”
- **Storage engine**: “how are changes represented and persisted?”

Some systems embed MVCC deeply into the storage engine; others separate it.

### Locks (logical) vs latches (physical)
- **Locks**: protect *logical* resources (row/table) for correctness across transactions.
- **Latches**: short-term mutual exclusion for *in-memory* structures (buffer pool hash table, B-tree page) to keep data structures consistent.

## Background work (why it exists)
DBs do significant work asynchronously to keep foreground latency low:
- flushing dirty pages
- checkpoints
- vacuum/garbage collection for MVCC
- statistics refresh
- index maintenance
- replication shipping/apply
- compaction (LSM)

## Recap
- A query flows through: protocol → parse/rewrite → optimize → execute.
- Execution is tightly coupled with transactions, concurrency control, and storage.
- Background workers handle maintenance to avoid doing everything in the critical path.

## Common misconceptions / pitfalls
- **“The optimizer always picks the best plan.”** It’s best according to its cost model and available stats.
- **“Concurrency control is just locks.”** Many DBs use MVCC plus locks (and latches) for different purposes.

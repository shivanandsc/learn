# Database Internals: Query Processing (Parser, Optimizer, Executor)

This module explains what happens inside the database when you run a query, focusing on SQL-style systems.

## 1) Parsing and binding
### Parsing
- Converts SQL text into an AST.
- Validates syntax.

### Binding / name resolution
- Maps identifiers to catalog objects (which `users` table? which `id` column?).
- Checks types and resolves functions/operators.

Output: a **bound** logical representation of the query.

## 2) Logical optimization (rewrites)
Rule-based transformations that preserve semantics:
- predicate pushdown
- projection pruning (read fewer columns)
- constant folding
- view expansion
- subquery rewrites (e.g., EXISTS → semi-join)

These make the later cost-based step easier and faster.

## 3) Cost-based optimization (planning)
The optimizer chooses among equivalent physical strategies.

### Relational algebra operators
- scan
- filter
- join
- group/aggregate
- sort

### Statistics
Decisions rely on estimates:
- row counts
- value distributions/histograms
- distinct counts
- correlation between columns (hard)

Bad stats ⇒ bad plans.

### Access path selection
For each table reference, pick:
- sequential scan
- index lookup
- index range scan

### Join order and join algorithms
Choosing join order is often the biggest win.

Common join algorithms:
- **Nested loop join**: good when one side is small and indexed.
- **Hash join**: good for large equi-joins; builds hash table.
- **Merge join**: good when inputs are already sorted.

### Plan caching
Many DBs cache compiled/planned queries (prepared statements), trading:
- faster repeated execution
- risk of plan staleness when data distribution changes

## 4) Execution engine
The chosen plan is executed by operators.

### Iterator model
Each operator exposes `next()` returning one row at a time.
- Simple, composable
- Higher per-row overhead

### Vectorized execution
Operators process batches (vectors) of rows.
- Better CPU cache usage
- Lower interpretation overhead

### Parallel execution
DBs may:
- parallelize scans
- parallelize joins/aggregations
- use multiple worker threads/processes

Parallelism must coordinate with memory limits and transaction isolation.

## 5) Writes (INSERT/UPDATE/DELETE)
Write queries often execute as:
- locate target rows (scan/index)
- apply changes
- update indexes
- emit WAL records
- update MVCC metadata / locks

## Recap
- Query processing is a pipeline: parse/bind → rewrite → cost-based planning → execution.
- Optimizer quality depends heavily on statistics.

## Common misconceptions / pitfalls
- **“Using an index is always best.”** For low selectivity predicates, a sequential scan can be cheaper.
- **“The optimizer can’t be wrong.”** It uses estimates; skew and correlations can mislead it.

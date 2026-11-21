# Hands-on Labs

> Six guided exercises with SQLite and PostgreSQL that let you observe pages, indexes, plans, transactions, versions, and the write-ahead log directly.

## Learning objectives

- Set up a safe local environment for experiments.
- Observe page size, index effects, and query plans.
- Watch MVCC row versions and snapshot isolation in action.
- Observe journal mode and WAL files.
- Reproduce a lock wait and a deadlock, then diagnose them.

## Prerequisites

- [Storage Engine and Pages](./04-storage-engine-and-pages.md)
- [Indexes](./05-indexes-btree-and-lsm.md)
- [Transactions](./07-transactions-and-concurrency.md)
- [WAL](./08-wal-and-crash-recovery.md)
- Tools: the `sqlite3` command-line shell, and Docker (for PostgreSQL).

## 1. Intuition

Reading about a combustion engine teaches you a lot, but you understand it better after taking one apart. These labs are your take-apart session. Each is small, safe, and ends with an "expected output" so you can tell whether your mental model matches reality.

Outputs shown are examples. Exact numbers, spacing, and wording vary by version; the **pattern** is what matters.

## 2. How it works

How to run each lab:

1. Read the goal and the question it answers.
2. Run the commands exactly, in order.
3. Compare to the expected output. If it differs, reread the related file instead of changing the commands.
4. Answer the reflection question before moving on.
5. Clean up so later labs start fresh.

Environment setup:

```bash
# SQLite (often preinstalled; otherwise install your platform's sqlite3 package)
sqlite3 --version

# PostgreSQL in a throwaway container
docker run --name pglab -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres:16
docker exec -it pglab psql -U postgres
```

Remove the container when done:

```bash
docker rm -f pglab
```

## 3. Architecture / data flow

```mermaid
flowchart LR
    L1["Lab 1 pages"] --> L2["Lab 2 indexes"]
    L2 --> L3["Lab 3 plans"]
    L3 --> L4["Lab 4 WAL"]
    L4 --> L5["Lab 5 MVCC"]
    L5 --> L6["Lab 6 locks and deadlock"]
```

Each box is a lab. They follow the order of the guide: storage, indexes, planning, logging, versions, and locking.

| Lab | Engine | Concept | Related file |
| --- | --- | --- | --- |
| 1 | SQLite | Pages and file size | [Storage](./04-storage-engine-and-pages.md) |
| 2 | SQLite | Index effect | [Indexes](./05-indexes-btree-and-lsm.md) |
| 3 | PostgreSQL | Plans and estimates | [Query Processing](./06-query-processing-and-optimizer.md) |
| 4 | SQLite | Journal and WAL | [WAL](./08-wal-and-crash-recovery.md) |
| 5 | PostgreSQL | MVCC versions | [Transactions](./07-transactions-and-concurrency.md) |
| 6 | PostgreSQL | Lock wait and deadlock | [Failure Modes](./11-failure-modes-and-troubleshooting.md) |

## 4. Internals deep dive

### Lab 1: Pages and file size (SQLite)

Goal: see that a database file is an array of fixed-size pages.

```bash
rm -f lab1.db
sqlite3 lab1.db
```

```sql
PRAGMA page_size;
PRAGMA page_count;
CREATE TABLE t(id INTEGER PRIMARY KEY, payload TEXT);
WITH RECURSIVE n(i) AS (SELECT 1 UNION ALL SELECT i + 1 FROM n WHERE i < 5000)
INSERT INTO t(payload) SELECT hex(randomblob(50)) FROM n;
PRAGMA page_count;
```

Expected output (values vary):

```text
4096
0
~190
```

A brand-new empty file may report 0 pages until the first table is created. Check that file size equals page_size times page_count:

```bash
ls -l lab1.db
```

```text
-rw-r--r--  1 you  staff  778240 ... lab1.db
```

Reflection: why is the size an exact multiple of 4096? (Because the file is an array of pages.)

### Lab 2: Index effect (SQLite)

```sql
CREATE TABLE users(id INTEGER PRIMARY KEY, email TEXT, country TEXT);
WITH RECURSIVE n(i) AS (SELECT 1 UNION ALL SELECT i + 1 FROM n WHERE i < 50000)
INSERT INTO users(email, country) SELECT 'u' || i || '@x.com', CASE WHEN i % 2 = 0 THEN 'IN' ELSE 'US' END FROM n;
.timer on
SELECT count(*) FROM users WHERE email = 'u49999@x.com';
CREATE INDEX idx_email ON users(email);
SELECT count(*) FROM users WHERE email = 'u49999@x.com';
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'u49999@x.com';
```

Expected: the count is 1 both times; the timed run after the index is noticeably faster (for example milliseconds versus fractions of a millisecond), and the plan shows `SEARCH users USING INDEX idx_email (email=?)`.

Reflection: what happens to the plan if you query `WHERE lower(email) = 'u49999@x.com'`? (It returns to a scan, because the function hides the column from the index.)

### Lab 3: Plans and estimates (PostgreSQL)

```sql
CREATE TABLE orders(id serial PRIMARY KEY, customer_id int, amount numeric);
INSERT INTO orders(customer_id, amount) SELECT (random() * 1000)::int, random() * 100 FROM generate_series(1, 200000);
EXPLAIN SELECT * FROM orders WHERE customer_id = 5;
ANALYZE orders;
EXPLAIN SELECT * FROM orders WHERE customer_id = 5;
CREATE INDEX ON orders(customer_id);
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 5;
```

Expected pattern: first plan is a `Seq Scan` with a possibly poor row estimate; after `ANALYZE` the estimate is near 200 rows; after indexing, a `Bitmap Heap Scan` or `Index Scan` appears with `actual` rows close to `rows`.

Reflection: which number tells you the statistics are healthy? (Estimated versus actual rows.)

### Lab 4: Journal and WAL (SQLite)

```bash
rm -f lab4.db*
sqlite3 lab4.db
```

```sql
PRAGMA journal_mode;
PRAGMA journal_mode=WAL;
CREATE TABLE log(x);
INSERT INTO log VALUES (1), (2), (3);
```

In a second terminal while the first is still open:

```bash
ls -l lab4.db*
```

Expected output of the two PRAGMA statements:

```text
delete
wal
```

The `ls` listing should show `lab4.db`, `lab4.db-wal`, and `lab4.db-shm`. The first PRAGMA reports the default mode (`delete`), the second reports `wal`. Now checkpoint:

```sql
PRAGMA wal_checkpoint(TRUNCATE);
```

```text
0|0|0
```

The WAL file shrinks to zero length because its contents were copied into the main database file.

Reflection: why can readers continue while the writer appends to the WAL? (Readers use the main file plus WAL records up to their snapshot point.)

### Lab 5: MVCC versions (PostgreSQL)

```sql
CREATE TABLE acct(id int PRIMARY KEY, bal int);
INSERT INTO acct VALUES (1, 100);
SELECT ctid, xmin, xmax, * FROM acct;
UPDATE acct SET bal = 50 WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM acct;
```

Expected pattern (transaction numbers differ):

```text
 ctid  | xmin | xmax | id | bal
-------+------+------+----+-----
 (0,1) |  741 |    0 |  1 | 100

 ctid  | xmin | xmax | id | bal
-------+------+------+----+-----
 (0,2) |  742 |    0 |  1 |  50
```

The visible row moved to slot 2 with a new `xmin`. Inspect dead versions using the optional extension:

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT lp, t_xmin, t_xmax FROM heap_page_items(get_raw_page('acct', 0));
```

```text
 lp | t_xmin | t_xmax
----+--------+--------
  1 |    741 |    742
  2 |    742 |      0
```

The old version (slot 1) is still physically present with `xmax` set to the updater. Run `VACUUM acct;` and look again: slot 1 becomes reusable.

### Lab 6: Lock wait and deadlock (PostgreSQL)

Use two `psql` sessions.

```sql
-- setup
CREATE TABLE r(id int PRIMARY KEY, v int);
INSERT INTO r VALUES (1, 0), (2, 0);
```

Session A, then Session B, then A, then B:

```sql
-- A
BEGIN; UPDATE r SET v = 1 WHERE id = 1;
-- B
BEGIN; UPDATE r SET v = 1 WHERE id = 2;
-- A (blocks, waiting for B)
UPDATE r SET v = 2 WHERE id = 2;
-- B (completes the cycle)
UPDATE r SET v = 2 WHERE id = 1;
```

Expected: after about one second (`deadlock_timeout`), one session reports:

```text
ERROR:  deadlock detected
DETAIL:  Process 120 waits for ShareLock on transaction 801; blocked by process 121.
Process 121 waits for ShareLock on transaction 800; blocked by process 120.
```

The other session's update then succeeds. Run `ROLLBACK;` in both.

Reflection: how would you avoid it? (Always update rows in the same order, such as ascending id.)

## 5. Worked example

Putting it together: a mini investigation.

1. In Lab 2, run `EXPLAIN QUERY PLAN` before and after the index.
2. Note the plan change from `SCAN` to `SEARCH`.
3. Conclude: an index cut the pages read from the whole table to a few.
4. Link to Lab 1: the full scan read every page counted by `page_count`.
5. Link to Lab 4: the insert load was logged before it was checkpointed into the main file.

Your written conclusion should read like: "Scanning 50,000 rows touched all N pages. The index reduced this to about three or four page reads because a B-tree has high fanout."

## 6. Tradeoffs and limitations

- SQLite is single-writer and embedded, so it demonstrates storage and logging well but not multi-process concurrency.
- PostgreSQL in Docker is realistic but adds Docker overhead; timings are not production benchmarks.
- Small tables may fit in cache and hide I/O effects; use large enough row counts.
- `pageinspect` requires privileges and is for learning, not production use.

## 7. Common misconceptions and pitfalls

- Forgetting `ANALYZE`, then misreading plans.
- Leaving a `BEGIN` open in a session, which causes confusing locks in later labs.
- Expecting identical numbers; sizes and timings vary.
- Running `EXPLAIN ANALYZE` on destructive statements; wrap them in `BEGIN` and `ROLLBACK`.
- Forgetting to remove the container, then hitting port conflicts.

## 8. Recap

- Database files are arrays of pages, and sizes are multiples of page size.
- An index changes the plan from a full scan to a tree search.
- Estimates versus actuals reveal statistics quality.
- MVCC leaves old row versions in place until vacuum.
- Deadlocks arise from opposite lock orders and are resolved by aborting one transaction.

## 9. Self-check questions

1. What does `PRAGMA page_count` times `PRAGMA page_size` give you?
2. What changed in the plan after adding an index?
3. Why does the old row version still exist after an update?
4. What does `PRAGMA wal_checkpoint` do?
5. How do you prevent the deadlock in Lab 6?

## Next

Continue to the [Cheat Sheet](./13-cheat-sheet.md).

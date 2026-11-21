# Database Internals: Indexes (B-Tree/B+Tree and LSM)

This module covers the two dominant index/storage designs used by modern databases: **B+trees** (classic disk-oriented) and **LSM trees** (write-optimized).

## What an index provides
An index accelerates access by organizing keys so the engine can:
- avoid scanning the whole table
- support efficient range queries (ordered indexes)

Costs:
- extra storage
- extra writes on inserts/updates/deletes
- possible contention under heavy write workloads

## B+tree indexes
Most relational databases use **B+trees** (often casually called B-trees).

### Structure
- **Internal nodes**: separator keys + child pointers
- **Leaf nodes**: (key → row pointer) entries, often linked as a leaf-level list

### Why it works well on disk
- High fan-out: each node holds many keys/pointers (page-sized).
- Tree height is small (often 2–4 levels for large datasets).

### Operations
- **Lookup**: root → internal nodes → leaf
- **Range scan**: find start key then follow leaf links sequentially
- **Insert**:
  - find leaf
  - insert if space else split leaf, propagate split upward
- **Delete**:
  - mark/delete entry
  - may merge/redistribute nodes (often deferred)

### Clustered vs secondary indexes
- **Clustered index**: leaf contains full rows (table stored as B+tree).
- **Secondary index**: leaf contains key + pointer (RID or primary key).

### Common performance characteristics
- Great for mixed read/write and range queries.
- Random inserts into the middle of the keyspace can cause frequent page splits.
- Requires careful locking/latching for concurrent updates.

## LSM trees (Log-Structured Merge)
LSM is widely used in write-heavy systems (many key-value stores, some relational engines’ storage layers).

### Core idea
Turn random writes into sequential writes:
- Writes go to an in-memory structure (**memtable**).
- Memtable is flushed to disk as an immutable sorted file (**SSTable**).
- Background **compaction** merges SSTables to keep read cost bounded.

### Components
- **Memtable**: in-memory sorted map (skiplist/tree)
- **WAL**: durability for memtable updates
- **SSTables**: sorted runs on disk
- **Bloom filters**: quickly test if a key is *not* in an SSTable
- **Compaction**: merge levels, discard overwritten/deleted keys

### Reads
A point lookup may check:
1) memtable
2) recent SSTables
3) older levels

Compaction + bloom filters reduce how many files are touched.

### Writes
- Typically fast and sequential.
- But can cause **write amplification** during compaction.

### Range scans
- Possible, but may need to merge iterators across multiple SSTables.
- Often slower/more complex than B+tree range scans.

## B+tree vs LSM: tradeoffs

| Dimension | B+tree | LSM |
|---|---|---|
| Write latency | can be higher (random I/O, splits) | often low (append + flush) |
| Read latency | predictable | may touch multiple levels/files |
| Range scans | excellent | more expensive (merge across files) |
| Space amplification | moderate | can be higher depending on compaction |
| Write amplification | lower/moderate | can be high (compaction) |

## Recap
- B+trees optimize reads and range queries with page-based navigation.
- LSM trees optimize writes by batching and merging in the background.

## Common misconceptions / pitfalls
- **“LSM is always faster.”** Great for writes, but read/range workloads and compaction pressure can hurt.
- **“B-tree height grows quickly.”** With page-sized nodes, height stays small even for large datasets.

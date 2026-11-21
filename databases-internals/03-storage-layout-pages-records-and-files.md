# Database Internals: Storage Layout (Files, Pages, Records)

This module explains how databases typically represent tables/indexes on disk and how that maps to I/O.

## Why storage layout matters
Disk (or SSD) is orders of magnitude slower than RAM. DBs therefore:
- read/write in **blocks/pages**
- cluster related data to reduce random I/O
- maintain metadata for fast navigation

Even “in-memory” databases often use page-like structures and logs for durability.

## Files and relations
A table (relation) and an index are usually backed by one or more files:
- **data file(s)**: hold table pages
- **index file(s)**: hold index pages
- **metadata/catalog**: schemas, stats, constraints
- **WAL/log files**: sequential append

Some DBs use a single tablespace file; others use many segment files.

## Pages (blocks)
A **page** is the unit of I/O and caching.

Typical page contents:
- page header (page id, LSN, flags)
- slot directory (for variable-length rows)
- free space
- row/record area

### Fixed vs variable-length rows
- Fixed-length records are simple and fast.
- Variable-length records need indirection (slot array) to keep row identifiers stable even if row size changes.

## Row identifiers
Common forms:
- **RID/TID**: (page_id, slot_id)
- **primary key**: logical identifier (may or may not map to physical location)

Many relational engines store secondary indexes pointing to the primary key (or RID), not to full rows.

## Heap-organized vs clustered storage

### Heap file (unordered)
- Rows are appended wherever there is space.
- Fast inserts.
- Reads may require indexes or full scans.

### Clustered by primary key
- Rows stored in key order (often a B+tree where leaf pages contain rows).
- Great for range scans by primary key.
- Inserts can cause page splits.

## Page splits and fragmentation
When an ordered structure (like B+tree) needs to insert into a full page:
- it **splits** the page into two
- updates parent pointers

Consequences:
- write amplification
- fragmentation (logical order differs from physical order)

DBs mitigate with fill factors, periodic maintenance, or compaction.

## Overflow and TOAST-like storage
Large columns (BLOB/TEXT) may be stored:
- out-of-row with a pointer from the main row
- compressed
- chunked across pages

## Checksums, LSN, and page validation
Many engines store:
- **checksum**: detect torn/corrupt pages
- **LSN (log sequence number)**: last WAL record applied to that page

LSN is crucial for recovery: if page LSN < redo LSN, it needs redo.

## Recap
- Data is stored in files made of pages.
- Rows are located via RIDs/slots or keys.
- Layout choices trade off insert speed, range scans, and fragmentation.

## Common misconceptions / pitfalls
- **“A row equals a file record.”** Rows move; slots/RIDs provide stable references.
- **“SSDs make layout irrelevant.”** SSDs reduce seek cost but caching, write amplification, and locality still matter.

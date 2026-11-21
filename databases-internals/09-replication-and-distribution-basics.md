# Database Internals: Replication and Distribution Basics

This module covers how databases replicate data and what changes when a database becomes distributed.

## Why replicate/distribute?
- **High availability**: survive node failures.
- **Read scaling**: serve reads from replicas.
- **Geo proximity**: lower latency.
- **Sharding**: store more data than one node can hold.

## Replication primitives

### Leader–follower (primary–replica)
- One node accepts writes (leader).
- Followers replicate changes.

Replication can be:
- **statement-based**: replicate SQL statements (simpler, can be nondeterministic pitfalls).
- **row-based**: replicate row changes (more robust, more data).
- **logical**: replicate logical change events.
- **physical**: replicate WAL/page changes.

### Synchronous vs asynchronous replication
- **Sync**: commit waits for replicas (stronger durability, higher write latency).
- **Async**: leader commits locally and ships later (risk of data loss on leader crash).

### Replication lag
Followers may be behind; consequences:
- stale reads
- failover may lose recent transactions (async)

## Consensus replication (quorums)
Some distributed DBs use quorum/consensus (e.g., Raft/Paxos-like ideas):
- writes require agreement from a majority
- improves correctness under failure
- adds coordination overhead

## Sharding (partitioning)
**Sharding** splits data across nodes.

Common strategies:
- **range sharding**: contiguous key ranges per shard (good for range scans, hotspot risk).
- **hash sharding**: distribute by hash(key) (good balance, range scans harder).

Key issues:
- distributed transactions across shards
- rebalancing shards
- cross-shard queries/joins

## Distributed transactions (very high-level)
To update multiple shards consistently:
- classic approach: **2-phase commit (2PC)**
- modern approach: consensus + careful design to reduce cross-shard transactions

Tradeoffs:
- coordination cost
- availability vs consistency under partitions

## CAP (as used in practice)
In a network partition, a system often chooses:
- **Consistency (C)**: reject some operations to avoid divergence
- **Availability (A)**: accept operations but risk conflicts/divergence

Real systems pick points along a spectrum with tunable consistency.

## Recap
- Replication improves availability and read scaling.
- Distribution introduces coordination, consensus, and cross-shard complexity.

## Common misconceptions / pitfalls
- **“Replication is a backup.”** Replication copies mistakes too; backups are separate.
- **“Async replication is safe enough.”** It can lose acknowledged writes on failover.

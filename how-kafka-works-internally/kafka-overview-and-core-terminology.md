# Kafka Overview & Core Terminology

## 1. TL;DR
Apache Kafka is a distributed, append-only log system for building high-throughput, low-latency event streaming pipelines and applications. Internally, Kafka stores events as ordered logs (partitions) replicated across brokers, with producers appending and consumers reading via offsets. Use Kafka when you need durable, scalable event ingestion, stream processing, and decoupled communication between services.

_Assumption/scope_: This note focuses on Kafka’s internal model and terminology (topics/partitions/replicas, producers/consumers, offsets, brokers, controllers).

## 2. Background & Motivation
Traditional messaging systems often emphasized queue semantics (messages removed once consumed) and broker-centric state. As event-driven architectures grew, systems needed:

- High throughput (MB/s to GB/s), achieved via sequential disk I/O and batching.
- Durability and replay (ability to reprocess events from the past).
- Horizontal scalability and fault tolerance.
- Fan-out (many consumers reading the same stream independently).

Kafka’s key shift is treating data as a distributed commit log where consumers track their own position (offset) rather than the broker “remembering what you consumed.”

## 3. Core Concepts

| Term | Meaning |
|---|---|
| Event / Record | A key-value message with headers and timestamp, stored durably in Kafka. |
| Topic | A named stream of records. |
| Partition | An ordered, append-only log within a topic; the unit of parallelism. |
| Offset | A monotonically increasing position of a record within a partition. |
| Broker | A Kafka server process that stores partitions and serves produce/fetch requests. |
| Cluster | A set of brokers working together. |
| Replica | A copy of a partition stored on a broker for fault tolerance. |
| Leader Replica | The replica that handles all reads/writes for a partition (in most configs). |
| Follower Replica | Replicas that replicate from the leader. |
| ISR (In-Sync Replicas) | Replicas caught up enough to be considered safe for acknowledging writes. |
| Producer | Client that publishes records to topics. |
| Consumer | Client that reads records from topics. |
| Consumer Group | A set of consumers sharing work; each partition is assigned to at most one consumer in a group. |
| Commit (offset commit) | Persisting the consumer’s position so it can resume after restart/rebalance. |
| Retention | Policy controlling how long data is kept (time/size), independent of consumption. |
| Log Compaction | Retention mode keeping the latest value per key (plus tombstones). |

### Build-up from first principles
- Kafka stores data as logs, not queues.
- A topic is split into partitions to scale and parallelize.
- Each partition is an ordered sequence of records; ordering is guaranteed only within a partition.
- Consumers read by offset and can replay by seeking to earlier offsets.

### Analogy (hardest idea: consumer-driven offsets)
Think of a public library (Kafka) with shelves (partitions). Books are placed in order with numbered slots (offsets). Readers (consumers) use bookmarks (committed offsets) to remember where they left off. The library doesn’t remove books when someone reads them; different readers can start at different slots and re-read old books.

## 4. Theory (Deep Dive)

### Architecture / data flow
```mermaid
flowchart LR
  P[Producer] -->|Produce requests| B1[Broker (Leader Partition)]
  B1 -->|Replication| B2[Broker (Follower)]
  B1 -->|Replication| B3[Broker (Follower)]
  C1[Consumer (Group A)] -->|Fetch requests| B1
  C2[Consumer (Group B)] -->|Fetch requests| B1
```

### Ordering and parallelism
- Partition is the atomic unit of ordering.
- Parallelism comes from having many partitions and many consumers in a group.
- Constraint: In a consumer group, each partition is processed by at most one consumer at a time.

### Retention vs consumption
Kafka does not delete records when consumed. Deletion is governed by:
- Time-based retention (e.g., keep 7 days)
- Size-based retention (keep last N GB)
- Compaction (keep latest per key)

### Performance characteristics (high level)
- Sequential writes (append-only) are fast.
- Batching and compression improve throughput.
- Reads are typically served from OS page cache if hot.

## 5. Practical Usage

### Minimal “hello world”
Create a topic and produce/consume a message.

```bash
# Create topic
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic demo --partitions 3 --replication-factor 1

# Produce
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic demo
# type: hello

# Consume (from beginning)
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --from-beginning
```

### Realistic example: consumer group scaling
```bash
# Start two consumers in the same group to split partitions
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --group g1
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demo --group g1
```
Explanation:
- With 3 partitions and 2 consumers, partitions are assigned across the consumers.
- Add a 3rd consumer to fully utilize all partitions.

## 6. Common Pitfalls & Best Practices

| Mistake | Why it happens | Fix |
|---|---|---|
| Expecting total order across a topic | Ordering is per-partition only | Use a single partition or a keying strategy to keep related events in one partition |
| Treating Kafka like a queue that deletes on consume | Kafka is a log with retention | Configure retention appropriately; design consumers to handle replay |
| Too few partitions | Limits throughput and consumer parallelism | Choose partitions based on expected throughput and consumer count |
| Too many partitions | Increases metadata, open files, recovery time | Scale partitions thoughtfully; benchmark |
| Forgetting to commit offsets properly | Leads to duplicates or data loss depending on strategy | Use at-least-once by committing after processing; consider transactions for EOS |

## 7. Comparison with Alternatives

| Approach | When it fits | Key trade-off |
|---|---|---|
| Traditional message queue (e.g., RabbitMQ) | Task distribution, per-message acknowledgments, complex routing | Often less optimized for long retention/replay and very high throughput logs |
| Kafka | Event streaming, replayable logs, fan-out, high throughput | More operational complexity; ordering only per partition |
| Cloud pub/sub (managed) | Want fully managed ops, simpler setup | Vendor-specific semantics; may differ in retention/replay guarantees |

## 8. Real-World Applications
- Event-driven microservices (domain events, integration events)
- Log aggregation and metrics pipelines
- Clickstream and behavioral analytics ingestion
- Stream processing backbones (Kafka Streams, Flink, Spark Structured Streaming)
- CDC (Change Data Capture) pipelines (e.g., Debezium → Kafka)

## 9. Practice Questions

### 9.1 Conceptual (5 questions)
1. 🟢 Why does Kafka use partitions, and what do partitions imply about ordering?
2. 🟢 What is an offset, and who “owns” it: the broker or the consumer?
3. 🟡 What is the difference between retention and consumer acknowledgment?
4. 🟡 Why can multiple consumer groups read the same topic independently?
5. 🔴 If you need strict ordering for a single entity (e.g., orderId), how should you partition the topic?

### 9.2 Applied / Scenario-Based (3–5 questions)
1. 🟡 You have 6 partitions and 10 consumers in one group. What happens to the extra consumers?
2. 🟡 A consumer restarts and reprocesses old messages. What are the likely offset/commit causes?
3. 🔴 You increase partitions from 10 to 30 for a keyed topic. What subtle behavior change can occur for key ordering?

### 9.3 Coding / Hands-On (2–3 tasks)
1. 🟢 Create a topic with 4 partitions and test how a consumer group of 2 divides work.
2. 🟡 Produce keyed messages and verify that the same key always goes to the same partition.

### 9.4 Interview-Style (3 questions)
1. 🟡 Explain Kafka’s log-based model and why it enables replay.
2. 🔴 Describe the trade-offs between many partitions vs few partitions.
3. 🔴 What guarantees can Kafka provide about ordering, durability, and delivery?

## 10. Answers & Explanations

<details>
<summary>Answer to 9.1 Q1</summary>

Partitions allow a topic to scale horizontally: different partitions can live on different brokers and be processed in parallel. Ordering is guaranteed only within a single partition; across partitions there is no global order.

</details>

<details>
<summary>Answer to 9.1 Q2</summary>

An offset is the position of a record within a partition. Conceptually the consumer owns its position (it decides what to read next); Kafka can store committed offsets on behalf of the consumer group, but the consumer controls when/what to commit.

</details>

<details>
<summary>Answer to 9.1 Q3</summary>

Retention controls how long Kafka keeps data regardless of whether it was consumed. Acknowledgment/commits control where a particular consumer group resumes reading. You can consume data multiple times as long as it hasn’t expired due to retention/compaction.

</details>

<details>
<summary>Answer to 9.1 Q4</summary>

Because consumption does not delete data, multiple groups can maintain independent offsets and read the same partitions at different rates and positions.

</details>

<details>
<summary>Answer to 9.1 Q5</summary>

Use a partitioning strategy that maps all events for a given entity key (e.g., orderId) to the same partition (typically by setting the record key). This preserves per-entity ordering while still allowing parallelism across different keys.

</details>

<details>
<summary>Answer to 9.2 Q1</summary>

Only up to 6 consumers can be active because each partition can be assigned to at most one consumer in the group. The remaining 4 consumers will be idle (assigned no partitions).

</details>

<details>
<summary>Answer to 9.2 Q2</summary>

Common causes: offsets were not committed (or committed too infrequently), commits happened before processing and a crash caused re-read (depending on when crash occurred), or the consumer group changed/rebalanced and resumed from the last committed offset. Also possible: the consumer used `auto.offset.reset=earliest` and there was no committed offset.

</details>

<details>
<summary>Answer to 9.2 Q3</summary>

If you rely on key-based partitioning, increasing partition count changes the mapping from key → partition (hash modulo partitionCount). That can break ordering for a key across the change because new records for the same key may land in a different partition than older records.

</details>

<details>
<summary>Answer to 9.3 Q1</summary>

When you run two consumers in the same group, Kafka assigns partitions across them (e.g., 2 partitions to one consumer and 2 to the other for a 4-partition topic). You can observe assignment via consumer logs or by noticing which consumer prints which records.

</details>

<details>
<summary>Answer to 9.3 Q2</summary>

Kafka’s default partitioner (and most client partitioners) uses the key to select a partition deterministically (typically via hashing). As long as the number of partitions is unchanged, the same key will always map to the same partition.

</details>

<details>
<summary>Answer to 9.4 Q1</summary>

Kafka persists records in an append-only log. Consumers track offsets, so they can restart and resume from a committed offset or intentionally seek backward to replay. This decouples storage from consumption.

</details>

<details>
<summary>Answer to 9.4 Q2</summary>

More partitions increases parallelism and throughput potential, but increases overhead: more files, more metadata, more replication work, longer leader elections and recovery, and more complex rebalances. Fewer partitions reduces overhead but can bottleneck producers/consumers.

</details>

<details>
<summary>Answer to 9.4 Q3</summary>

Ordering: guaranteed per partition. Durability: depends on replication factor and `acks`/ISR settings; with replication and appropriate acks, data survives broker failure. Delivery: typically at-least-once with standard commits; exactly-once requires idempotent producer + transactions and careful processing.

</details>

## 11. Summary / Cheat Sheet
- Kafka is a distributed log: topics → partitions → ordered records.
- Ordering is per-partition only.
- Consumers track offsets; data deletion is controlled by retention/compaction.
- Consumer groups scale processing: one consumer per partition at a time.
- Partitions are the main lever for throughput and parallelism.

## 12. Further Reading
- Apache Kafka Documentation (official)
- Kafka: The Definitive Guide (O’Reilly)
- Kafka Improvement Proposals (KIPs) for deeper internals and protocol evolution

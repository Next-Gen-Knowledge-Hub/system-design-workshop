# 12. Stream processing

Companion notes for **Chapter 12** of *Designing Data-Intensive Applications*
(2nd ed.). Batch jobs have an end of file. **Streams** do not: events
keep arriving. A stream processor is a batch job that never finishes,
which changes time, state, and failure.

## What this chapter is for

Streams are how you:

- Move data between systems continuously (not a nightly dump).
- Keep **derived** stores (search, cache, analytics, ML features) nearly
  fresh.
- React to patterns (“this card was used in two countries in 10 minutes”).

The input is **unbounded**. You never “rerun the whole file” unless you
kept the log and can **replay**.

## Transmitting event streams

### Messaging systems (AMQP / JMS style)

A broker **assigns messages to consumers**. When a consumer acks, the
message is often **deleted** (or marked done). Good as **async RPC** /
task queue: emails, thumbnail jobs, “call this webhook.” Competing
consumers scale workers; they scramble global order.

Weak when you needed to:

- Replay last week after a bug.
- Let a **new** consumer see history.
- Keep a strict order across the whole topic.

Some brokers offer retention; the classic mental model is still “queue
of work items,” not “system of record.”

### Log-based message brokers

Kafka, Pulsar, Kinesis-like systems: a **sharded append-only log**.
Messages stay for a retention period (or until compacted). Consumers
**checkpoint an offset**. Order is **per partition**. Parallelism =
partition count (ch. 7).

This is a cousin of the replication log (ch. 6) and LSM segments
(ch. 4), and leaders per partition sit near consensus ideas (ch. 10).

Use this when the stream *is* data: activity events, CDC, event
sourcing, several independent consumers (search indexer, warehouse
loader, fraud, notifications) each with their own offset. A new
consumer can rewind. A buggy consumer can fix code and reprocess.

**Compaction:** keep the latest value per key (changelog). Replay
becomes snapshot-ish + tail — the same shape as a replicated state
feed.

## Databases and streams

If the OLTP database is the system of record, other systems need its
changes — continuously.

### Keeping systems in sync

**Dual write** (app writes DB *and* Kafka in one request) looks easy and
**loses** consistency: one of the two writes fails, or they reorder, or
the process dies between them. Don’t.

Better patterns: **CDC**, or **transactional outbox** (business rows +
outbox row in one DB transaction; publisher drains outbox). Both make
the database commit the source of truth for “this change exists.”

### Change data capture

Tail the database’s replication / WAL log, emit a changelog (Debezium-
style connectors are the common industrial form). The log is extracted
from what the DB already durable-wrote for replication (ch. 6).

CDC turns “keep Elastic / cache / warehouse honest” into **consume and
apply**. Rebuild = replay from a snapshot + log, or from a compacted
topic.

### State, streams, and immutability

**Event sourcing** (ch. 3): the log *is* the system of record; tables
are derived. CDC is the shy version: the table stays the record, the
log is extracted.

**Immutability:** an event happened. Corrections are new events (or
tombstones), not silent edits in the middle of history — except when
law forces erasure (ch. 14), which needs an explicit tombstone /
crypto-shred / rewrite policy.

Immutability also helps concurrency: streams define an order per shard;
consumers apply in that order. Multi-shard facts need extra care
(ch. 13).

## Processing streams

### Uses of stream processing

- Complex event processing / pattern detection (fraud, ops alerts).
- Windowed analytics (count per minute, rolling unique users).
- Materialized views (keep a query result up to date as events arrive).
- Enrichment (order event + current user profile).
- Feeding search, caches, and feature stores with low delay.

### Reasoning about time

**Event time** — when it happened in the world (embedded in the
payload). **Processing time** — when your operator saw it. They diverge
under lag, retries, and partitions.

**Late events (stragglers)** arrive after you thought a window was
closed. **Watermarks** are a bet: “we believe we have seen everything
with event time ≤ T.” Too aggressive → wrong aggregates; too relaxed →
high latency. Always say which clock a window uses.

**Windows:** tumbling, hopping / sliding, session. Session windows
follow gaps in activity, not fixed walls.

### Stream joins

| Kind | Picture | State you keep |
|---|---|---|
| Stream–stream | Two event types in a time window | Events in the window |
| Stream–table | Event + current dimension row | Table as compacted changelog |
| Table–table | Two evolving relations | Both changelogs |

The “table” in stream land is often a **changelog compacted into local
state** (RocksDB / LSM beside the operator — ch. 4). Partitioning must
align join keys or you shuffle.

### Fault tolerance

If an operator dies:

1. Restore **operator state** from a checkpoint / savepoint.
2. **Rewind** the source log to the matching offset.
3. Replay; side effects must be **idempotent** or transactional.

“Exactly-once” *inside* the engine means: replay + idempotent state
updates + transactional sinks (or equivalent). **End-to-end**
exactly-once with an external database still needs idempotent writes,
dedupe keys, or a transactional protocol (Kafka transactions + careful
sinks; or outbox patterns from ch. 8).

In interviews, **at-least-once + idempotent handlers** is the design
you can defend. Exactly-once is a property you name with its boundary
(which systems are inside the fence).

## How this shows up when you design something

- Notifications / emails: task queue may be enough if you will never
  replay.
- Activity pipeline, search indexer, fraud, warehouse tail: log-based
  broker.
- “Keep Elasticsearch in sync”: CDC or outbox, not dual write.
- Metrics per minute: windowed aggregation; say event time + watermark
  policy.
- End-to-end sketch: produce to a partitioned log → process with
  rewindable offsets → idempotent sink (or transactional outbox).

## Check yourself

1. Why does deleting a message on ack make a new analytics consumer
   unable to catch up?
2. Dual-write vs CDC: which failure mode does CDC remove?
3. Event time vs processing time for “orders in the last 5 minutes”
   during a consumer lag spike — which clock should the window use?
4. What do you store in operator state for a stream–table join?
5. At-least-once delivery + non-idempotent “insert payment” handler:
   what goes wrong on replay?
6. Why is order guaranteed per Kafka partition but not across the
   topic?
7. Log compaction: what is preserved, what is gone, and who wants it?
8. Watermark too far ahead vs too far behind: which error do you get
   in each case?

Continue to [A philosophy of streaming systems](../13-streaming-philosophy/).

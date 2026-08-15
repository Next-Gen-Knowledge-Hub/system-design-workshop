# 12. Stream processing

Companion notes for **Chapter 12** of *Designing Data-Intensive Applications*
(2nd ed.). Batch jobs have an end of file. **Streams** do not: events
keep arriving. A stream processor is a batch job that never finishes,
which changes time, state, and failure.

This is the DDIA theory behind
[kafka-workshop](../../kafka-workshop). Read them together.

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
message is **deleted**. Good as **async RPC** / task queue (work items,
emails). Bad if you needed to:

- Replay last week after a bug.
- Let a new consumer see history.
- Keep a strict global order.

Competing consumers scale workers; they scramble order.

### Log-based message brokers

Kafka, Pulsar, Kinesis-like: a **sharded append log**. Messages stay on
disk for a retention period (or compacted). Consumers **checkpoint an
offset**. Order is **per partition**. Parallelism = partitions (ch. 7).
This is a cousin of the replication log (ch. 6) and LSM segments (ch. 4),
and it is a form of consensus on the log (ch. 10).

Use this when the stream *is* data: activity events, CDC, event
sourcing, several independent consumers (search indexer, warehouse
loader, fraud, notifications) each with their own offset.

[kafka-workshop overview](../../kafka-workshop/1-setup/OVERVIEW.md)
is the worked example.

## Databases and streams

If the OLTP database is the system of record, other systems need its
changes.

**Dual write** (app writes DB *and* Kafka) looks easy and **loses**
consistency: one of the two writes fails, or they reorder. Don’t.

**Change data capture (CDC):** tail the database’s replication log,
emit a changelog. Debezium-style. The log is the truth; the stream is a
copy. **Log compaction** (keep the latest value per key) makes the
stream replayable as a snapshot + tail — Kafka compacted topics,
changelog streams in Kafka Streams.

**Event sourcing** (ch. 3): the log *is* the system of record; the SQL
table is derived. CDC is the shy version: the table stays the record,
the log is extracted.

**Immutability:** an event happened. Corrections are new events, not
edits in the middle of history (except GDPR — ch. 1 / 14 — which
forces tombstones / crypto-erase / rewrite policies).

Keeping caches, search, and warehouses in sync is then: **consume the
changelog, apply.** Rebuild = replay from the start (or from a
snapshot). That is the “Unix pipeline” of ch. 11, left running.

## Processing streams

**Uses:** CEP (pattern detection), windowed analytics (count per minute),
materialized views (keep a table up to date), joining streams to
tables (enrich order with user).

**Time:** **event time** (when it happened in the world) vs **processing
time** (when your job saw it). They diverge. Late events (**stragglers**)
arrive after you already closed a window. Watermarks are a bet: “we
think we have seen everything up to T.” Too aggressive, you miss; too
relaxed, you wait.

**Windows:** tumbling, hopping, sliding, session. Always say which
clock.

**Stream joins:**

- Stream–stream (two event types in a window).
- Stream–table (event + current user profile).
- Table–table as two changelogs.

The table in stream land is a **changelog compacted to state**, often
in a local RocksDB (LSM, ch. 4) next to the operator.

**Fault tolerance:** if an operator dies, restore **state** from a
checkpoint and **rewind the log** to that offset. Exactly-once *inside
the engine* is “replay + idempotent state updates + transactional
sinks.” End-to-end exactly-once with an external DB still needs
idempotent writes or a transactional protocol
([kafka exactly-once](../../kafka-workshop/3-internals-and-reliability/RELIABILITY.md)).
At-least-once + idempotent handlers is the design you can actually
explain in an interview.

## How this shows up when you design something

- Notifications: task queue (AMQP style) is enough if you don’t replay.
- Activity pipeline, search indexer, fraud: log-based broker.
- “Keep Elasticsearch in sync”: CDC, not dual write.
- Metrics per minute: windowed aggregation; say event time + watermark.
- Capstone shape: see
  [kafka order pipeline](../../kafka-workshop/8-capstone-order-pipeline/).

## Ties to other workshops

- Almost all of [kafka-workshop](../../kafka-workshop), especially
  [stream concepts](../../kafka-workshop/6-stream-processing/CONCEPTS.md).
- [WAL / segmented log](../../distributed-systems-workshop/2-replication/README_Log.md)

## Check yourself

1. Why does deleting a message on ack make a new analytics consumer
   unable to catch up?
2. Dual-write vs CDC: which failure mode does CDC remove?
3. Event time vs processing time for “orders in the last 5 minutes”
   during a consumer lag spike.
4. What do you store in operator state for a stream–table join?

Continue to [A philosophy of streaming systems](../13-streaming-philosophy/).

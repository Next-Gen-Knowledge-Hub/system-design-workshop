# Stream processing

These notes teach stream processing from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What stream processing is

A batch job reads a file that ends. A stream does not end. Payment events keep arriving after you have already emitted an answer. A stream processor is a batch computation that never finishes, and that changes three things: which clock you trust, what state you keep between events, and what a retry is allowed to do.

Streams are how you move data between systems continuously, rather than dumping a file once a night. They keep derived stores nearly fresh: search, a cache, an analytics table, machine-learning features. They also react to a pattern as it forms. "This card was used in two countries in ten minutes" is not a question you ask tomorrow. It is a question over events that are still arriving.

The input is **unbounded**. There is no whole file to rerun unless you kept the log and can **replay** it.

## How events are transmitted

### A broker that hands out work

In the AMQP and JMS style, a broker **assigns** each message to a consumer. When the consumer acknowledges it, the broker often **deletes** the message, or marks it done. This is a good **async RPC**, a task queue. Send an email. Generate a thumbnail. Call this webhook. **Competing consumers** share the queue and scale the workers. They also scramble global order: two workers pull two jobs, and the jobs finish in whichever order the workers finish, not in the order they were enqueued.

That model is weak when you later need one of three things. You need to replay last week after a bug. You need a **new** consumer to see history it was not alive for. You need a strict order across the whole topic, not just inside one worker's batch. Some of these brokers can retain messages. The mental model is still a queue of work items, not a system of record. Once the ack deletes the message, an analytics job you deploy next month has nothing to catch up on.

### An append-only log

Kafka, Pulsar, and Kinesis-like systems are a **sharded append-only log**. A producer appends. The broker keeps the record for a retention period, or until compaction removes older values. A consumer does not receive the message as an assignment that then vanishes. It **checkpoints an offset**: "I have read this partition through position N." Another consumer group has its own offsets. Order holds **per partition**, not across the topic. Two partitions are two independent logs. Parallelism equals the number of partitions. Add consumers beyond that count and the extras sit idle, because each partition is read by one consumer in the group.

This log is a cousin of two older structures. A database **replication log** is the ordered history a leader ships so replicas can replay it. An **LSM** store (a log-structured merge tree) writes recent changes as immutable segments, and compaction later keeps the latest version of each key. A partition leader that orders the appends is also close to **consensus**: the nodes have to agree on a single history for that partition, and on who the leader is, even when a node fails.

Use a log when the stream **is** the data. Activity events, change data capture, and event sourcing all fit. So does a set of independent consumers on one topic: a search indexer, a warehouse loader, a fraud detector, a notification sender. Each group keeps its own offset and reads at its own pace. A new consumer can rewind. A buggy consumer can be fixed and can reprocess.

**Compaction** keeps the latest value per key and drops the older ones. The topic becomes a **changelog**: the history of updates to a row, folded toward current state. Replaying it is then "build a snapshot from the compacted log, then follow the tail." That is the same shape as a replicated feed of state, where a follower catches up by applying the latest record per key and then stays on the live end.

What compaction preserves is the latest value of each key. What it throws away is every earlier value of that key. The consumers who want that trade are the ones materializing current state. The consumers who need every historical event, such as an audit of each click, want a retained log, not a compacted one.

## From a database to a stream

If the OLTP database is the system of record, every other system needs its changes, continuously, not in last night's dump.

### Two writes in one request

**Dual write** means the application, inside one request, writes the database and also writes Kafka. It looks like one operation. It is two. One of them fails and the other succeeds. The process dies after the database commit and before the produce. The two writes land in different orders at different readers. The database and the log then disagree about whether the change exists. Do not do this.

Two patterns make the database commit the source of truth for "this change exists."

A **transactional outbox** writes the business rows and an outbox row in the **same** database transaction. A publisher process reads the outbox and produces to the log. If the publisher crashes, the outbox row is still there to be drained. The change cannot be committed in the database and absent from the outbox, because they committed together.

**Change data capture** does not ask the application to write a second row. It tails the database's replication log, the **WAL** (write-ahead log): the durable record the database already wrote so its own replicas can replay. A connector in the style of Debezium turns that log into a changelog on the broker. "Keep Elasticsearch, the cache, and the warehouse honest" becomes: consume the changelog and apply it. Rebuilding one of those stores is a replay from a snapshot plus the log, or a replay from a compacted topic.

### The log as the record, or as an extract

**Event sourcing** makes the log itself the system of record. Tables and views are derived from it by replay. Change data capture is the shy version of the same idea. The table stays the record. The log is extracted from what the database already durable-wrote for replication.

**Immutability** means an event, once written, happened. A correction is a new event, or a tombstone that says "this key is gone." It is not a silent edit in the middle of the history. The exception is when the law forces erasure of a person. That needs an explicit policy: a tombstone, crypto-shredding (destroy the key that decrypts that person's bytes), or a rewrite of the log. A quiet overwrite of an old offset is not a policy.

Immutability also gives consumers an order to apply. Within one shard, the log defines the sequence, and each consumer applies it in that sequence. A fact that spans shards needs extra care. The card used in Germany and the card used in Japan may land on different partitions. Nothing in the broker orders those two partitions against each other. A detector that assumes one global sequence will mis-order them.

## Processing events that do not stop

Typical jobs on this machinery:

- Complex event processing: detect a pattern, such as fraud or an operational alert.
- Windowed analytics: a count per minute, or a rolling count of unique users.
- A materialized view: keep the result of a query up to date as events arrive, instead of recomputing it from scratch.
- Enrichment: take an order event and attach the user's current profile.
- Feeding search, caches, and feature stores with a short delay.

### Which clock the window uses

**Event time** is when the event happened in the world, carried in the payload. **Processing time** is when your operator happened to see it. They diverge whenever there is lag, a retry, or a slow partition.

Say the job is "orders in the last five minutes." A consumer falls twenty minutes behind. If the window uses processing time, those orders land in "now," the five minutes when the operator woke up, and the dashboard shows a spike that did not happen. If the window uses event time, each order falls in the five minutes when the customer paid, which is the question you asked. Say which clock you are using. Do not leave it implicit.

A **late event**, a straggler, arrives after you thought its window was already closed. A **watermark** is a bet: "we believe we have seen every event with event time ≤ T." Advance T aggressively and you close the window before the stragglers arrive, so the aggregate is wrong. Hold T back and you wait a long time to emit, so latency goes up. A watermark set too far ahead produces the wrong total. A watermark set too far behind produces a late but more complete total.

Three window shapes:

- A **tumbling** window cuts time into fixed, non-overlapping buckets. Each event belongs to one bucket. 12:00–12:05, then 12:05–12:10.
- A **hopping** or **sliding** window has a fixed size and overlaps the next one. A five-minute window that advances every minute puts one event into several windows.
- A **session** window follows a gap in activity, not a wall on the clock. The session ends after the card has been quiet for thirty minutes, however long the burst before that gap was.

### Joining two things that move

| Kind | What meets | State you keep |
|---|---|---|
| Stream–stream | Two event types inside a time window | The events that are still inside the window |
| Stream–table | An event plus the current row of a dimension | The table, as a compacted changelog |
| Table–table | Two relations that both change | Both changelogs |

A stream–stream join of authorizations and captures keeps both for as long as they sit inside the window, then matches them. When the window expires, those events can leave the state.

A stream–table join is the enrichment case. The order is the stream. The user profile is the table. The operator does not wait for a window. It looks up whatever the profile is **now**. That "now" is the changelog of the profile topic, compacted into **local state** beside the operator, often RocksDB, an LSM store on the worker's disk. Replay of the compacted topic rebuilds the table. Partitioning has to put the same join key on the same worker. If orders are partitioned by order id and profiles by user id, matching rows never meet unless you shuffle one side onto the other's key.

A table–table join keeps both changelogs. When either side updates, the join result updates.

### When the operator dies

Failure recovery is three steps, in this order.

1. Restore **operator state** from a checkpoint or a savepoint: the counts, the open windows, the local copy of the table.
2. **Rewind** the source log to the offset that matches that checkpoint.
3. Replay. Anything the replay does to the outside world has to be **idempotent** or transactional, so doing it twice is the same as doing it once.

**Exactly-once inside the engine** means that combination: replay, plus state updates that tolerate being applied again, plus transactional sinks (or an equivalent). The fence is the engine. **End-to-end** exactly-once, with an external database on the other side of the sink, still needs one of: idempotent writes, dedupe keys, or a transactional protocol. Kafka transactions with a careful sink are one such protocol. An outbox is another. The name "exactly-once" is incomplete until you say which systems sit inside the fence.

The design you can defend is **at-least-once delivery plus idempotent handlers**. The log will redeliver after a crash. The handler must absorb that. Consider a handler that inserts a payment row with a fresh database identity on every message, and that does not remember it already inserted this payment. Replay inserts a second payment. The customer is charged twice. An idempotent handler keys the insert on a stable payment id, so the second attempt finds the row and stops.

Notifications and email are the case where a task queue is enough, if you will never replay. An activity pipeline, a search indexer, fraud detection, and a warehouse tail want the log, because those consumers appear, break, and reprocess. Keeping Elasticsearch in sync is change data capture or an outbox, not a dual write. A metric per minute is a windowed aggregation, and the design names the event-time clock and the watermark policy. The path from end to end is: produce onto a partitioned log, process with offsets you can rewind, and sink with an idempotent write or a transactional outbox.

## Recap

A stream has no end of file, so the processor keeps state, names its clock, and recovers by restoring a checkpoint and replaying. A queue that deletes on ack is a task list; a retained, partitioned log is data you can rewind, with order inside each partition and a separate offset per consumer. Compaction keeps the latest value per key and drops the rest, which is what a table-shaped consumer wants. The database, not a second write beside it, should be what makes a change real: an outbox row in the same transaction, or a changelog tailed from the write-ahead log. Windows on event time, with an explicit watermark, survive lag that would corrupt a processing-time count. Exactly-once is real only inside a stated boundary; outside it, at-least-once plus an idempotent sink is what stops a replay from inserting the payment twice.

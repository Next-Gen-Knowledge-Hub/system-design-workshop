# Design trade-offs cheat sheet

A one-screen reminder of the choices DDIA spends chapters on. Use this when
you are sketching a design and need to name the cost of each pick. Details
live in the chapter folders.

Source: *Designing Data-Intensive Applications*, 2nd ed. (Kleppmann &
Riccomini). This is a study aid, not a copy of the book.

---

## Architecture (ch. 1–2)

| Choice | You gain | You pay |
|---|---|---|
| One big OLTP database for everything | Simple ops, transactions | Analytics crush production; wrong storage layout |
| Separate warehouse / lake | OLTP stays fast; analysts get scans | ETL lag; two copies of truth to keep honest |
| Cloud managed service | Elasticity, less ops toil | Cost at steady load, less tunability, vendor lock-in |
| Self-host | Control, often cheaper at steady load | You own paging, upgrades, capacity |
| Distribute early | Geo latency, fault isolation | Partial failure, clocks, consensus |
| Stay on one machine | Determinism, simpler debugging | Ceiling on size, no region failover |

## Data models (ch. 3)

| Model | Fits | Hurts |
|---|---|---|
| Relational | Joins, many-to-many, constraints | Rigid schema migrations; object-relational mismatch |
| Document | Nested, self-contained records | Cross-document joins; duplication |
| Graph | Many hops, "who is connected to whom" | Point-get OLTP; analytics scans |
| Event log + CQRS | Audit, rebuild views, complex domain | Querying the log directly is painful |
| DataFrame / arrays | ML, stats, wide numeric tables | Not an OLTP source of truth |

## Storage (ch. 4)

| Engine | Writes | Reads |
|---|---|---|
| LSM-tree (Cassandra, RocksDB, Lucene) | Fast sequential appends | Reads may merge several files; compaction CPU |
| B-tree (Postgres, InnoDB, most RDBMS) | Update-in-place pages | Predictable point/range reads |
| Column store | Bulk load | Huge scans / aggregations; bad for single-row OLTP |
| Vector index (HNSW / IVF) | Approximate nearest neighbor | Not exact; extra pipeline for embeddings |

## Encoding (ch. 5)

| Format | Evolution | Cost |
|---|---|---|
| Language-native pickle / Java serialization | Fragile, often unsafe | Fast to write, terrible to share |
| JSON / XML / CSV | Ubiquitous, human-readable | Vague types, verbose |
| Protobuf / Avro | Explicit backward/forward rules | Need a schema story |

## Replication (ch. 6)

| Style | Consistency | Failure story |
|---|---|---|
| Single-leader, sync | Stronger durability | Leader wait; write unavailable if follower down |
| Single-leader, async | Fast writes | Lag; failover can lose acknowledged writes |
| Failover without fencing | — | Split-brain: two leaders, diverging data |
| Multi-leader | Write locally in several regions | Conflict resolution, causality bugs |
| Leaderless (quorum) | Survives some nodes down | Last-write-wins or siblings; not linearizable by default |

## Sharding (ch. 7)

| Scheme | Good at | Bad at |
|---|---|---|
| Hash of key | Even load, simple point gets | Range scans fan out |
| `hash % node_count` | Looks simple | Adding a node reshuffles almost every key |
| Fixed shard count / consistent hash | Grow by moving whole shards | Must pick a shard count (or ring) you can live with |
| Key range | Range queries, locality | Hot ranges (celebrity keys, sequential ids) |
| Local secondary index | Writes stay on one shard | Query fans out to every shard |
| Global secondary index | Query hits few shards | Writes touch many shards; often async |

## Transactions (ch. 8)

| Isolation | Prevents | Still allows |
|---|---|---|
| Read committed | Dirty reads/writes | Read skew, lost updates, write skew |
| Snapshot isolation | Read skew, most phantoms | Write skew (often); lost updates depend on impl |
| Serializable (2PL / SSI / serial exec) | The anomalies in the table | Throughput / latency cost; 2PL stalls |
| Cross-system XA 2PC | Atomic commit across products | Coordinator blocking; avoid as a habit |
| DB-internal distributed txn | Cross-shard atomicity in one product | Cross-shard latency |
| Outbox + idempotent consume | Exactly-once side effects without XA | Async lag; you own dedupe keys |

## Distributed reality (ch. 9–10)

| You wish | Reality |
|---|---|
| Timeout means the peer is dead | Or the packet is slow, or you are slow |
| Wall clocks order events | NTP steps, pauses, unbounded skew |
| A lock in Redis is enough | Need fencing tokens or you get zombies |
| Replication looks linearizable | Only if the algorithm actually is (usually consensus) |
| Linearizable everything | Geo cost; refuse ops in a minority partition |
| 50-node etcd for user traffic | Consistent core stays small (3–5 voters) |

## Derived data (ch. 11–13)

| Style | Bounded input | Unbounded input |
|---|---|---|
| Batch | Rerun the job, debug, backfill | Too slow for "now" |
| Stream | — | Continuously update derived views |
| Dual write app→DB and app→Kafka | Looks simple | Split brain on partial failure |
| CDC / outbox from system of record | One truth; replayable | Pipeline lag; schema evolution |
| Dual (lambda batch+stream) | Recompute when logic changes | Two pipelines to keep equivalent |
| Kappa (replay the log) | One codepath | Log must be rich enough to rebuild |

Continue with the chapter folders in order, starting at
[ch. 1](./1-architecture-tradeoffs/).

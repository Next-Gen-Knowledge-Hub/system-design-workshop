# Design trade-offs cheat sheet

A one-screen reminder of the choices the two books spend chapters on.
DDIA tables name the cost of a *data-system* pick. Hard Parts tables
(at the bottom) name the cost of a *service-seam* pick. Do not fuse a
row from one book into a chapter of the other. Details:
[`INDEX.md`](./INDEX.md).

Source: *Designing Data-Intensive Applications*, 2nd ed. (Kleppmann &
Riccomini) and *Software Architecture: The Hard Parts* (Ford, Richards,
Sadalage, Dehghani). This is a study aid, not a copy of either book.

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

Continue with the DDIA folders in [`ddia/`](./ddia/), starting at
[ch. 1](./ddia/1-architecture-tradeoffs/). Hard Parts starts at
[hard-parts ch. 1](./hard-parts/1-no-best-practices/).

---

## Hard Parts (service seams — do not merge into the tables above)

Source: *Software Architecture: The Hard Parts* (Ford, Richards,
Sadalage, Dehghani, 2021). Same words as DDIA, different job: **where
you cut**, not **how the engine replicates**. Details:
[`INDEX.md`](./INDEX.md). Folders: [`hard-parts/`](./hard-parts/).

### Coupling and the quantum (HP 2–3)

| Choice | You gain | You pay |
|---|---|---|
| One quantum (one deployable, one data) | Simple ops, one transaction | Change and scale move together |
| Many quanta | Independent deploy, scale, failure | Static + dynamic coupling across the wire |
| Split services, keep one database | Looks like microservices | You did not split the quantum; the DB is still the unit |

### Decomposition (HP 4–5)

| Choice | You gain | You pay |
|---|---|---|
| Component-based extraction | Incremental; you can measure coupling | Slow; leftover shared kernel |
| Tactical fork (copy the monolith, delete half) | Fast isolation | Two codebases to starve or merge later |

### Operational data split (HP 6)

| Choice | You gain | You pay |
|---|---|---|
| Keep the monolith DB | Joins, one backup | Schema change is a company meeting |
| Split by data domain, then polyglot | Independent life cycle | Distributed access (HP 10) and ownership (HP 9) |

DDIA [ch. 3](./ddia/3-data-models/) / [ch. 4](./ddia/4-storage-and-retrieval/) pick
an *engine*. Hard Parts 6 decides *whether the engine is still shared*.

### Granularity (HP 7)

| Force that **splits** | Force that **joins** |
|---|---|
| Volatility, independent scale, fault isolation, security boundary, extension | Single DB transaction, chatty workflow, shared code, inseparable data |

### Reuse (HP 8)

| Pattern | You gain | You pay |
|---|---|---|
| Copy the snippet | No shared release | Drift |
| Shared library | One fix | Version hell; deploy coupling |
| Shared service | One runtime | Availability and latency on the critical path |
| Sidecar / mesh | Cross-cutting without a domain service | Platform tax |

### Ownership and "consistency" (HP 9–10)

| Choice | You gain | You pay |
|---|---|---|
| Single owner of a table | Clear writes | Other services must ask or replicate |
| Joint ownership | Teams unblocked | Who is allowed to write? |
| Interservice call for a join | Fresh data | Runtime coupling, outages cascade |
| Replicate a column / cache | Local reads | Staleness; invalidation |

This is **not** DDIA replica consistency ([ch. 6](./ddia/6-replication/)) and
**not** linearizability ([ch. 10](./ddia/10-consistency-and-consensus/)). It
is "which service is allowed to `UPDATE` this row, and how do others
see it?"

### Workflow and sagas (HP 11–12)

| Choice | You gain | You pay |
|---|---|---|
| Orchestrator | Visible state, easier timeout/compensate | Orchestrator is a coupling hub |
| Choreography | No central boss | History is scattered; harder to debug |
| Atomic saga (try to look like one commit) | Simpler mental model | Often a 2PC in disguise — see [DDIA 8](./ddia/8-transactions/) |
| Eventual saga + compensations | Survives partial failure | You own the undo story |

A saga is an **application protocol**. Isolation levels are a **database
protocol**. Do not paste HP 12 into DDIA 8.

### Contracts (HP 13) vs encoding (DDIA 5)

| Choice | You gain | You pay |
|---|---|---|
| Strict schema (types, required fields) | Breaks at generate/build time | Change coordination |
| Loose contract (schemaless bag) | Easy to add fields | Stamp coupling; consumers break in prod |
| Fat event (stamp the world) | Consumer needs no extra calls | Bandwidth; accidental coupling to internals |

DDIA 5 is **how bytes evolve** (Avro/Protobuf rules). Hard Parts 13 is
**how much of the domain you leak** across a service boundary.

### Analytical (HP 14) vs OLAP (DDIA 1, 11)

| Choice | You gain | You pay |
|---|---|---|
| Central warehouse / lake | One place to query | Coupling every domain to a platform team |
| Data mesh (domain-owned products) | Scale ownership | You now operate a mesh of products, not a lake ticket |

---

DDIA tables above still start at [ch. 1](./ddia/1-architecture-tradeoffs/).
Hard Parts tables start at [hard-parts ch. 1](./hard-parts/1-no-best-practices/).

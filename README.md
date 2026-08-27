# System Design Workshop

A workshop for learning how to **design data-intensive systems**: picking storage,
replication, sharding, consistency, and processing models on purpose instead of
by buzzword.

We are guiding this workshop with **Martin Kleppmann and Chris Riccomini's
*Designing Data-Intensive Applications* (2nd edition, O'Reilly, 2026)** — the
book that sits at the center of this field. Each numbered folder is one chapter
of that book: the topics from the chapter, written as a human-friendly companion
you can read after (or alongside) the book, not as a substitute for it.

Each numbered folder is one chapter: a `README.md` that explains the
ideas in plain language, with check-yourself questions to lock them in.

The notes here are **original study material**. They do not copy the book. Read
the chapter, then use the matching folder to lock in the vocabulary, the
trade-offs, and how the idea shows up in a real design.

Later sections of this workshop will add:

- **Interview-style system design** — timed walkthroughs in the spirit of
 Alex Xu's *System Design Interview* volumes (not a theory replacement for
 DDIA).
- **More books in this field** — Vitillo, Burns, Ford/Richards (*The Hard
 Parts*), and others. DDIA stays the spine; those books fill interview
 drills, platform patterns, and staff-level trade-off language.

This repository contains the following topics

**Part A — Foundations (how to even talk about a design)**

1. [Trade-offs in data systems architecture](./1-architecture-tradeoffs/) — DDIA ch. 1
 - Operational vs analytical systems (OLTP / OLAP)
 - Cloud vs self-hosting, cloud-native architecture
 - Distributed vs single-node
 - Data systems, law, and society
2. [Defining nonfunctional requirements](./2-nonfunctional-requirements/) — DDIA ch. 2
 - Case study: social-network home timelines
 - Performance (latency, percentiles, SLAs)
 - Reliability and fault tolerance
 - Scalability and maintainability

**Part B — Data on one machine (models, disks, bytes)**

3. [Data models and query languages](./3-data-models/) — DDIA ch. 3
 - Relational vs document
 - Graphs, GraphQL, event sourcing / CQRS
 - DataFrames, matrices, arrays
4. [Storage and retrieval](./4-storage-and-retrieval/) — DDIA ch. 4
 - LSM-trees vs B-trees
 - Column stores, warehouses, vector indexes
5. [Encoding and evolution](./5-encoding-and-evolution/) — DDIA ch. 5
 - JSON / Protobuf / Avro
 - Dataflow through DBs, RPC, workflows, events

**Part C — Distributed data (the heart of system design)**

6. [Replication](./6-replication/) — DDIA ch. 6
 - Single-leader (sync/async, failover, fencing, replication logs)
 - Lag and session guarantees; multi-leader; sync engines / local-first
 - Leaderless quorums, conflicts, version vectors
7. [Sharding](./7-sharding/) — DDIA ch. 7
 - Hash vs range, hot spots, rebalancing, request routing
 - Multitenancy; local vs global secondary indexes
8. [Transactions](./8-transactions/) — DDIA ch. 8
 - ACID; read committed, snapshot, write skew, lost updates
 - Serializability (serial exec, 2PL, SSI); 2PC and exactly-once
9. [The trouble with distributed systems](./9-distributed-trouble/) — DDIA ch. 9
 - Partial failure; networks, timeouts, clocks, process pauses
 - Quorums, fencing, Byzantine faults; formal methods / Jepsen
10. [Consistency and consensus](./10-consistency-and-consensus/) — DDIA ch. 10
 - Linearizability (cost, CAP); logical clocks and ID generators
 - Consensus, replicated logs, coordination services (etcd/ZK)

**Part D — Derived data (how systems stay in sync without one giant DB)**

11. [Batch processing](./11-batch-processing/) — DDIA ch. 11
 - Unix pipelines; object stores; orchestration vs schedulers
 - MapReduce → dataflow; shuffle/skew; ETL, ML, serving views
12. [Stream processing](./12-stream-processing/) — DDIA ch. 12
 - Queues vs log-based brokers; CDC vs dual-write
 - Event time, watermarks, joins, exactly-once boundaries
13. [A philosophy of streaming systems](./13-streaming-philosophy/) — DDIA ch. 13
 - Derive don’t dual-write; batch+stream; unbundling the DB
 - End-to-end idempotency, async constraints, trust-but-verify

**Part E — Responsibility**

14. [Doing the right thing](./14-doing-the-right-thing/) — DDIA ch. 14
 - Predictive bias, accountability, feedback loops
 - Privacy, surveillance, erasure vs logs, law

**Part F — Coming later (not DDIA)**

15. [Interview-style system designs](./15-interview-designs/) — Xu Vol. 1 / Vol. 2 and our own designs
16. [Further reading in this field](./16-further-reading/) — Vitillo, Burns, Hard Parts, …

Cross-cutting: [Design trade-offs cheat sheet](./DESIGN_TRADEOFFS.md) — one
page per big choice (leader vs leaderless, B-tree vs LSM, hash vs range, …).

## How to use this workshop

1. Read the DDIA chapter (the book is the source of truth).
2. Read the matching folder here. It is written to be read, not skimmed: topics
 follow the book's headings, in everyday language.
3. Answer the **Check yourself** questions in your own words. A good
 answer has three parts: the takeaway, the failure mode it prevents, and
 one production example from a system you have worked on.

Work through the folders in order. Later chapters quietly assume earlier ones:
you cannot talk honestly about consensus (10) until you have felt replication
lag (6) and network ambiguity (9).

## First edition vs second edition

If you have notes from the **2017 first edition**, the chapter numbers moved.
This workshop follows the **2026 second edition** (the PDF in this repo).

| 1st edition | 2nd edition |
|---|---|
| Ch. 1 Reliable, scalable, maintainable | Split into ch. 1 (architecture trade-offs) + ch. 2 (nonfunctional requirements) |
| Ch. 2 Data models | Ch. 3 (+ GraphQL, event sourcing, DataFrames) |
| Ch. 3 Storage and retrieval | Ch. 4 (+ vector indexes) |
| Ch. 4 Encoding and evolution | Ch. 5 (+ durable execution / workflows) |
| Ch. 5 Replication | Ch. 6 (+ sync engines, local-first) |
| Ch. 6 Partitioning | Ch. 7 *Sharding* |
| Ch. 7 Transactions | Ch. 8 |
| Ch. 8 Trouble with distributed systems | Ch. 9 (+ formal methods) |
| Ch. 9 Consistency and consensus | Ch. 10 (largely rewritten) |
| Ch. 10 Batch processing | Ch. 11 (rewritten; MapReduce no longer the center) |
| Ch. 11 Stream processing | Ch. 12 |
| Ch. 12 The future of data systems | Split into ch. 13 (streaming philosophy) + ch. 14 (ethics) |

Feel free to use and make any change ;)

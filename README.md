# System Design Workshop

A workshop for learning how to **design data-intensive systems and how to
split them without lying about the cost**: picking storage, replication,
sharding, consistency, and processing models on purpose, then deciding
service granularity, data ownership, sagas, and contracts when the system
is no longer one process.

We are guiding this workshop with two books. They are kept on **separate
tracks**. Where a topic appears in both, the notes do not merge the
chapters — they **mention** the other track and send you there.

| Track | Folder | Book |
|---|---|---|
| **Data systems (DDIA)** | [`ddia/`](./ddia/) | Martin Kleppmann and Chris Riccomini, *Designing Data-Intensive Applications* (2nd edition, O'Reilly, 2026). |
| **Hard Parts** | [`hard-parts/`](./hard-parts/) | Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani, *Software Architecture: The Hard Parts* (O'Reilly, October 2021). Subtitle: *Modern trade-off analysis for distributed architectures*. |

Each book lives in its own track folder. DDIA chapters stay numbered
1–14 **inside** `ddia/`. Hard Parts is 1–15 inside `hard-parts/`.

Each numbered folder is one chapter of that book: the topics from the
chapter, rewritten as a human-friendly companion you can read after (or
alongside) the book.

The notes here are **original study material**. They do not copy the
books. Read the chapter, then use the matching folder to lock in the
vocabulary, the trade-offs, the failure mode, and one production example.
Keep the PDFs next to your other books; they are not committed in this
git repo.

The topic index for both books lives in [`INDEX.md`](./INDEX.md).
Cross-cutting choices live in [`DESIGN_TRADEOFFS.md`](./DESIGN_TRADEOFFS.md).

DDIA stays the data-systems spine; Hard Parts is the staff-level *split
and reassemble* lens, not a replacement.

## What this workshop assumes

You can write a service that talks to a database. You do not need prior
distributed-systems theory, microservice taxonomy, or saga catalogs —
that is what the folders are for. Where a chapter needs a concept from
an earlier one *in the same book*, it says so. Where the *other book*
covers the same word from a different angle, you get a one-line **See
also** and a link, not a rewrite.

## How the two books divide the work

Kleppmann and Riccomini is **how data systems behave**: OLTP vs OLAP,
encodings, replication, sharding, isolation, partial failure, consensus,
batch and stream, ethics of data about people. You finish it able to
*name the store and the consistency story*.

Ford, Richards, Sadalage, and Dehghani is **how you pull a monolith apart
and pay for the seams**: architecture quantum, modularity drivers,
decomposition, polyglot operational data, service granularity, reuse
(library vs service vs sidecar), data ownership, distributed access,
orchestration vs choreography, saga patterns, contracts, analytical data
mesh, and a method for trade-off analysis. You finish it able to *split
a system and argue the coupling you just bought*.

Read DDIA first if "quorum" and "write skew" are still fog. Read Hard
Parts first if you already operate a database and the pain is *where to
cut the services*. Either order works; the index is the map. Do not merge
a Hard Parts chapter into a DDIA folder because they share a word
("transaction", "consistency", "schema", "workflow", "OLAP").

This repository contains the following topics

### Track A — Data systems (*Designing Data-Intensive Applications*, 2e)

**Part A — Foundations (how to even talk about a design)**

1. [Trade-offs in data systems architecture](./ddia/1-architecture-tradeoffs/) — DDIA ch. 1
 - Operational vs analytical systems (OLTP / OLAP)
 - Cloud vs self-hosting, cloud-native architecture
 - Distributed vs single-node
 - Data systems, law, and society
 - **See also:** [Hard Parts no best practices](./hard-parts/1-no-best-practices/), [analytical data](./hard-parts/14-analytical-data/)
2. [Defining nonfunctional requirements](./ddia/2-nonfunctional-requirements/) — DDIA ch. 2
 - Case study: social-network home timelines
 - Performance (latency, percentiles, SLAs)
 - Reliability and fault tolerance
 - Scalability and maintainability
 - **See also:** [Hard Parts modularity drivers](./hard-parts/3-modularity/)

**Part B — Data on one machine (models, disks, bytes)**

3. [Data models and query languages](./ddia/3-data-models/) — DDIA ch. 3
 - Relational vs document
 - Graphs, GraphQL, event sourcing / CQRS
 - DataFrames, matrices, arrays
 - **See also:** [Hard Parts operational data / database types](./hard-parts/6-operational-data/)
4. [Storage and retrieval](./ddia/4-storage-and-retrieval/) — DDIA ch. 4
 - LSM-trees vs B-trees
 - Column stores, warehouses, vector indexes
5. [Encoding and evolution](./ddia/5-encoding-and-evolution/) — DDIA ch. 5
 - JSON / Protobuf / Avro
 - Dataflow through DBs, RPC, workflows, events
 - **See also:** [Hard Parts contracts](./hard-parts/13-contracts/)

**Part C — Distributed data (the heart of system design)**

6. [Replication](./ddia/6-replication/) — DDIA ch. 6
 - Single-leader (sync/async, failover, fencing, replication logs)
 - Lag and session guarantees; multi-leader; sync engines / local-first
 - Leaderless quorums, conflicts, version vectors
 - **See also:** [Hard Parts distributed data access](./hard-parts/10-distributed-data-access/)
7. [Sharding](./ddia/7-sharding/) — DDIA ch. 7
 - Hash vs range, hot spots, rebalancing, request routing
 - Multitenancy; local vs global secondary indexes
8. [Transactions](./ddia/8-transactions/) — DDIA ch. 8
 - ACID; read committed, snapshot, write skew, lost updates
 - Serializability (serial exec, 2PL, SSI); 2PC and exactly-once
 - **See also:** [Hard Parts data ownership / sagas](./hard-parts/9-data-ownership/), [transactional sagas](./hard-parts/12-transactional-sagas/)
9. [The trouble with distributed systems](./ddia/9-distributed-trouble/) — DDIA ch. 9
 - Partial failure; networks, timeouts, clocks, process pauses
 - Quorums, fencing, Byzantine faults; formal methods / Jepsen
10. [Consistency and consensus](./ddia/10-consistency-and-consensus/) — DDIA ch. 10
 - Linearizability (cost, CAP); logical clocks and ID generators
 - Consensus, replicated logs, coordination services (etcd/ZK)
 - **See also:** [Hard Parts eventual-consistency patterns](./hard-parts/9-data-ownership/) (application-level, not consensus)

**Part D — Derived data (how systems stay in sync without one giant DB)**

11. [Batch processing](./ddia/11-batch-processing/) — DDIA ch. 11
 - Unix pipelines; object stores; orchestration vs schedulers
 - MapReduce → dataflow; shuffle/skew; ETL, ML, serving views
 - **See also:** [Hard Parts analytical data / mesh](./hard-parts/14-analytical-data/)
12. [Stream processing](./ddia/12-stream-processing/) — DDIA ch. 12
 - Queues vs log-based brokers; CDC vs dual-write
 - Event time, watermarks, joins, exactly-once boundaries
 - **See also:** [Hard Parts workflows](./hard-parts/11-distributed-workflows/)
13. [A philosophy of streaming systems](./ddia/13-streaming-philosophy/) — DDIA ch. 13
 - Derive don’t dual-write; batch+stream; unbundling the DB
 - End-to-end idempotency, async constraints, trust-but-verify

**Part E — Responsibility**

14. [Doing the right thing](./ddia/14-doing-the-right-thing/) — DDIA ch. 14
 - Predictive bias, accountability, feedback loops
 - Privacy, surveillance, erasure vs logs, law

### Track B — Hard Parts (*Software Architecture: The Hard Parts*, 2021)

**Part B1 — Pulling things apart**

1. [No best practices](./hard-parts/1-no-best-practices/) — HP ch. 1
    - Why architecture is trade-offs, not recipes; ADRs; fitness functions
    - Architecture vs design; the book's running ticketing case
    - **See also:** [DDIA architecture forks](./ddia/1-architecture-tradeoffs/)
2. [Coupling](./hard-parts/2-coupling/) — HP ch. 2
    - Architecture quantum; static vs dynamic coupling; independent deploy
3. [Modularity](./hard-parts/3-modularity/) — HP ch. 3
    - Drivers: maintainability, testability, deployability, scale, availability
    - **See also:** [DDIA nonfunctional requirements](./ddia/2-nonfunctional-requirements/)
4. [Decomposition](./hard-parts/4-decomposition/) — HP ch. 4
    - Is the codebase decomposable? Afferent/efferent, main sequence
    - Component-based split vs tactical forking
5. [Component-based decomposition patterns](./hard-parts/5-component-patterns/) — HP ch. 5
    - Identify, gather, flatten, dependencies, domains, domain services
6. [Pulling apart operational data](./hard-parts/6-operational-data/) — HP ch. 6
    - Data disintegrators vs integrators; five-step split; database types
    - **See also:** [DDIA data models](./ddia/3-data-models/), [storage](./ddia/4-storage-and-retrieval/)
7. [Service granularity](./hard-parts/7-service-granularity/) — HP ch. 7
    - Disintegrators (scope, volatility, scale, fault, security, extension)
    - Integrators (transactions, workflow, shared code, data relationships)

**Part B2 — Putting things back together**

8. [Reuse patterns](./hard-parts/8-reuse-patterns/) — HP ch. 8
    - Replication, shared library, shared service, sidecar / mesh
9. [Data ownership and distributed transactions](./hard-parts/9-data-ownership/) — HP ch. 9
    - Single / common / joint ownership; eventual-consistency patterns
    - **See also:** [DDIA transactions](./ddia/8-transactions/) (isolation inside one DB)
10. [Distributed data access](./hard-parts/10-distributed-data-access/) — HP ch. 10
    - Interservice call, column replication, replicated cache, data domain
    - **See also:** [DDIA replication](./ddia/6-replication/)
11. [Managing distributed workflows](./hard-parts/11-distributed-workflows/) — HP ch. 11
    - Orchestration vs choreography; who owns workflow state
    - **See also:** [DDIA streams](./ddia/12-stream-processing/) (brokers and CDC, not saga style)
12. [Transactional sagas](./hard-parts/12-transactional-sagas/) — HP ch. 12
    - Atomic vs eventual × orchestrated vs choreographed; compensating updates
    - **See also:** [DDIA 2PC / exactly-once](./ddia/8-transactions/)
13. [Contracts](./hard-parts/13-contracts/) — HP ch. 13
    - Strict vs loose; stamp coupling
    - **See also:** [DDIA encoding and evolution](./ddia/5-encoding-and-evolution/)
14. [Managing analytical data](./hard-parts/14-analytical-data/) — HP ch. 14
    - Warehouse, lake, data mesh
    - **See also:** [DDIA OLTP/OLAP](./ddia/1-architecture-tradeoffs/), [batch](./ddia/11-batch-processing/)
15. [Build your own trade-off analysis](./hard-parts/15-trade-off-analysis/) — HP ch. 15
    - Entangled dimensions; MECE; out-of-context trap; bottom line over slideware

Cross-cutting: [topic index](./INDEX.md) · [design trade-offs cheat sheet](./DESIGN_TRADEOFFS.md).

## How to use this workshop

1. Read the book chapter. The book is the source of truth.
2. Read the matching folder here. Headings follow that book's topics.
3. When a **See also** points at the other track, follow it only if you
   need that angle (how the *store* behaves vs how you *cut the services*).
   Do not merge the two chapters in your notes.
4. Answer the **Check yourself** questions in your own words. A good
   answer has three parts: the takeaway, the failure mode it prevents, and
   one production example from a system you have worked on.

Work through each track in order the first time. You cannot talk honestly
about consensus (DDIA 10) until you have felt replication lag (6) and
network ambiguity (9). You cannot pick a saga (Hard Parts 12) until you
have named ownership (9) and workflow style (11).

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

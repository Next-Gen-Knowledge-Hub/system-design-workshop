# Topic index

Two books, two tracks. This page is the **index of topics as the books
organize them**, plus a cross-book map so you do not merge chapters that
happen to share a word.

- **DDIA** = *Designing Data-Intensive Applications*, 2e (Kleppmann & Riccomini) → [`ddia/`](./ddia/) numbered folders `1`–`14`
- **HP** = *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021) → [`hard-parts/`](./hard-parts/)

When a term appears in both, pick the job you actually have:

- DDIA folder — *how the data system behaves* (storage engine, isolation, replication, logs)
- Hard Parts folder — *how you cut services and pay for the seams* (quantum, ownership, saga style, contract tightness)

Follow the link you need; do not collapse the folders into one note.

---

## Cross-book map (same word, different job)

| Topic | DDIA (how the store behaves) | Hard Parts (how you split the system) |
|---|---|---|
| Trade-offs as the method | [D1 architecture forks](./ddia/1-architecture-tradeoffs/), [D2 NFRs](./ddia/2-nonfunctional-requirements/) | [H1 no best practices, ADRs, fitness](./hard-parts/1-no-best-practices/), [H15 analysis method](./hard-parts/15-trade-off-analysis/) |
| Scale / availability | [D2](./ddia/2-nonfunctional-requirements/) | [H3 modularity drivers](./hard-parts/3-modularity/), [H7 granularity](./hard-parts/7-service-granularity/) |
| Data models / which database | [D3 models](./ddia/3-data-models/), [D4 storage engines](./ddia/4-storage-and-retrieval/) | [H6 polyglot operational data](./hard-parts/6-operational-data/) (when to *split* the DB, then pick types) |
| Schema / contracts | [D5 encoding and evolution](./ddia/5-encoding-and-evolution/) | [H13 strict vs loose contracts, stamp coupling](./hard-parts/13-contracts/) |
| Replication / copies | [D6 replication](./ddia/6-replication/) (leaders, quorums, lag) | [H10 access patterns](./hard-parts/10-distributed-data-access/) (call vs replicate a column vs cache) |
| Sharding | [D7](./ddia/7-sharding/) | [H6](./hard-parts/6-operational-data/) data domains, not hash rings |
| Transactions | [D8 ACID, isolation, 2PC](./ddia/8-transactions/) | [H9 ownership + application eventual consistency](./hard-parts/9-data-ownership/), [H12 sagas](./hard-parts/12-transactional-sagas/) |
| Consistency | [D10 linearizability, consensus](./ddia/10-consistency-and-consensus/) | [H9 background / request / event sync](./hard-parts/9-data-ownership/) (not Raft) |
| Workflow / async | [D5 durable execution], [D12 streams, CDC](./ddia/12-stream-processing/), [D13 derive-don't-dual-write](./ddia/13-streaming-philosophy/) | [H11 orchestration vs choreography](./hard-parts/11-distributed-workflows/) |
| Analytical data | [D1 OLTP/OLAP](./ddia/1-architecture-tradeoffs/), [D11 batch](./ddia/11-batch-processing/) | [H14 warehouse / lake / mesh](./hard-parts/14-analytical-data/) |
| Coupling | Partial failure [D9](./ddia/9-distributed-trouble/) | Architecture quantum, static vs dynamic [H2](./hard-parts/2-coupling/) |

---

## Track A — *Designing Data-Intensive Applications* (by chapter)

Folders 1–14 live in [`ddia/`](./ddia/). Headings in each `README.md` follow the
2e chapter.

### DDIA 1 — Trade-offs in data systems architecture
[`ddia/1-architecture-tradeoffs`](./ddia/1-architecture-tradeoffs/)

- OLTP vs OLAP; cloud vs self-host; distributed vs single-node; law and society

### DDIA 2 — Defining nonfunctional requirements
[`ddia/2-nonfunctional-requirements`](./ddia/2-nonfunctional-requirements/)

- Timelines case; latency/percentiles; reliability; scalability; maintainability

### DDIA 3 — Data models and query languages
[`ddia/3-data-models`](./ddia/3-data-models/)

- Relational, document, graph/GraphQL, event sourcing/CQRS, DataFrames

### DDIA 4 — Storage and retrieval
[`ddia/4-storage-and-retrieval`](./ddia/4-storage-and-retrieval/)

- LSM vs B-tree; column stores; vector indexes

### DDIA 5 — Encoding and evolution
[`ddia/5-encoding-and-evolution`](./ddia/5-encoding-and-evolution/)

- JSON / Protobuf / Avro; dataflow through DBs, RPC, workflows, events

### DDIA 6 — Replication
[`ddia/6-replication`](./ddia/6-replication/)

- Single-leader, lag, multi-leader, leaderless, sync engines

### DDIA 7 — Sharding
[`ddia/7-sharding`](./ddia/7-sharding/)

- Hash vs range, rebalancing, routing, secondary indexes

### DDIA 8 — Transactions
[`ddia/8-transactions`](./ddia/8-transactions/)

- ACID; isolation anomalies; serializability; 2PC; exactly-once / outbox

### DDIA 9 — The trouble with distributed systems
[`ddia/9-distributed-trouble`](./ddia/9-distributed-trouble/)

- Partial failure, timeouts, clocks, fencing, Byzantine, Jepsen

### DDIA 10 — Consistency and consensus
[`ddia/10-consistency-and-consensus`](./ddia/10-consistency-and-consensus/)

- Linearizability, CAP, logical clocks, consensus, etcd/ZK

### DDIA 11 — Batch processing
[`ddia/11-batch-processing`](./ddia/11-batch-processing/)

- Pipelines, object stores, dataflow, shuffle, ETL / serving views

### DDIA 12 — Stream processing
[`ddia/12-stream-processing`](./ddia/12-stream-processing/)

- Brokers vs queues; CDC vs dual-write; event time; exactly-once boundaries

### DDIA 13 — A philosophy of streaming systems
[`ddia/13-streaming-philosophy`](./ddia/13-streaming-philosophy/)

- Derive don't dual-write; unbundling; idempotency; trust-but-verify

### DDIA 14 — Doing the right thing
[`ddia/14-doing-the-right-thing`](./ddia/14-doing-the-right-thing/)

- Bias, privacy, erasure vs logs, law

---

## Track B — *Software Architecture: The Hard Parts* (by chapter)

### HP 1 — What happens when there are no “best practices”?
[`hard-parts/1-no-best-practices`](./hard-parts/1-no-best-practices/)

- Why "the hard parts"; timeless advice; data in architecture decisions
- ADRs; architecture fitness functions
- Architecture vs design; Sysops Squad as the running case

### HP 2 — Discerning coupling
[`hard-parts/2-coupling`](./hard-parts/2-coupling/)

- Architecture quantum (independently deployable, cohesive)
- Static coupling; dynamic quantum coupling

### HP 3 — Architectural modularity
[`hard-parts/3-modularity`](./hard-parts/3-modularity/)

- Drivers: maintainability, testability, deployability, scalability, availability

### HP 4 — Architectural decomposition
[`hard-parts/4-decomposition`](./hard-parts/4-decomposition/)

- Is it decomposable? Afferent/efferent, abstractness, instability, main sequence
- Component-based decomposition vs tactical forking

### HP 5 — Component-based decomposition patterns
[`hard-parts/5-component-patterns`](./hard-parts/5-component-patterns/)

- Identify and size; gather common domain; flatten; dependencies; domains; domain services
- Fitness functions for governance

### HP 6 — Pulling apart operational data
[`hard-parts/6-operational-data`](./hard-parts/6-operational-data/)

- Data disintegrators vs integrators
- Five-step split of a monolith database
- Database types (relational, KV, document, column family, graph, NewSQL, cloud-native, time-series)

### HP 7 — Service granularity
[`hard-parts/7-service-granularity`](./hard-parts/7-service-granularity/)

- Disintegrators: scope, volatility, throughput, fault tolerance, security, extensibility
- Integrators: transactions, workflow, shared code, data relationships

### HP 8 — Reuse patterns
[`hard-parts/8-reuse-patterns`](./hard-parts/8-reuse-patterns/)

- Code replication; shared library; shared service; sidecars and service mesh
- When reuse actually adds value; platforms

### HP 9 — Data ownership and distributed transactions
[`hard-parts/9-data-ownership`](./hard-parts/9-data-ownership/)

- Single / common / joint ownership; split, domain, delegate, consolidate
- Distributed transactions; background / orchestrated-request / event-based consistency

### HP 10 — Distributed data access
[`hard-parts/10-distributed-data-access`](./hard-parts/10-distributed-data-access/)

- Interservice communication; column schema replication; replicated caching; data domain

### HP 11 — Managing distributed workflows
[`hard-parts/11-distributed-workflows`](./hard-parts/11-distributed-workflows/)

- Orchestration vs choreography; workflow state; coupling of the state owner

### HP 12 — Transactional sagas
[`hard-parts/12-transactional-sagas`](./hard-parts/12-transactional-sagas/)

- Three-letter saga catalog (atomic/eventual × orchestrated/choreographed × parallel-ish)
- Compensating updates; saga state machines

### HP 13 — Contracts
[`hard-parts/13-contracts`](./hard-parts/13-contracts/)

- Strict vs loose; microservices contracts; stamp coupling

### HP 14 — Managing analytical data
[`hard-parts/14-analytical-data`](./hard-parts/14-analytical-data/)

- Warehouse; lake; data mesh (product quantum, coupling)

### HP 15 — Build your own trade-off analysis
[`hard-parts/15-trade-off-analysis`](./hard-parts/15-trade-off-analysis/)

- Entangled dimensions; qualitative vs quantitative; MECE; out-of-context trap
- Model relevant cases; bottom line; snake oil

Appendices A–C in the book are reference lists, not workshop folders.

---

## Alphabetical index

| Term | Where |
|---|---|
| ADR (architecture decision record) | [H1](./hard-parts/1-no-best-practices/) |
| Afferent / efferent coupling | [H4](./hard-parts/4-decomposition/) |
| Analytical / OLAP | [D1](./ddia/1-architecture-tradeoffs/), [H14](./hard-parts/14-analytical-data/) |
| Architecture quantum | [H2](./hard-parts/2-coupling/) |
| Atomicity / ACID | [D8](./ddia/8-transactions/) |
| Availability | [D2](./ddia/2-nonfunctional-requirements/), [H3](./hard-parts/3-modularity/) |
| Avro / Protobuf / JSON | [D5](./ddia/5-encoding-and-evolution/) |
| B-tree vs LSM | [D4](./ddia/4-storage-and-retrieval/) |
| Batch processing | [D11](./ddia/11-batch-processing/) |
| CDC / outbox | [D8](./ddia/8-transactions/), [D12](./ddia/12-stream-processing/) |
| Choreography | [H11](./hard-parts/11-distributed-workflows/), [H12](./hard-parts/12-transactional-sagas/) |
| Column store | [D4](./ddia/4-storage-and-retrieval/), [H6](./hard-parts/6-operational-data/) (column *family* as a product type) |
| Compensating transaction | [H12](./hard-parts/12-transactional-sagas/) |
| Consensus / Raft / Paxos | [D10](./ddia/10-consistency-and-consensus/) |
| Contracts (strict / loose) | [H13](./hard-parts/13-contracts/) |
| CQRS / event sourcing | [D3](./ddia/3-data-models/) |
| Data lake | [H14](./hard-parts/14-analytical-data/) |
| Data mesh | [H14](./hard-parts/14-analytical-data/) |
| Data ownership | [H9](./hard-parts/9-data-ownership/) |
| Data warehouse | [H14](./hard-parts/14-analytical-data/), [D1](./ddia/1-architecture-tradeoffs/) |
| Decomposition (code) | [H4](./hard-parts/4-decomposition/), [H5](./hard-parts/5-component-patterns/) |
| Deployability | [H3](./hard-parts/3-modularity/) |
| Distributed vs single-node | [D1](./ddia/1-architecture-tradeoffs/) |
| Document model | [D3](./ddia/3-data-models/), [H6](./hard-parts/6-operational-data/) |
| Dual write | [D12](./ddia/12-stream-processing/), [D13](./ddia/13-streaming-philosophy/) |
| Encoding / schema evolution | [D5](./ddia/5-encoding-and-evolution/), [H13](./hard-parts/13-contracts/) |
| Eventual consistency (app-level) | [H9](./hard-parts/9-data-ownership/) |
| Eventual consistency (replicas) | [D6](./ddia/6-replication/) |
| Fitness function | [H1](./hard-parts/1-no-best-practices/), [H5](./hard-parts/5-component-patterns/) |
| Granularity (services) | [H7](./hard-parts/7-service-granularity/) |
| Graph database | [D3](./ddia/3-data-models/), [H6](./hard-parts/6-operational-data/) |
| Isolation levels | [D8](./ddia/8-transactions/) |
| Linearizability | [D10](./ddia/10-consistency-and-consensus/) |
| Maintainability | [D2](./ddia/2-nonfunctional-requirements/), [H3](./hard-parts/3-modularity/) |
| MECE list | [H15](./hard-parts/15-trade-off-analysis/) |
| Modularity | [H3](./hard-parts/3-modularity/) |
| NewSQL / cloud-native DB | [H6](./hard-parts/6-operational-data/) |
| OLTP | [D1](./ddia/1-architecture-tradeoffs/) |
| Orchestration (workflow) | [H11](./hard-parts/11-distributed-workflows/) |
| Orchestration (batch jobs) | [D11](./ddia/11-batch-processing/) |
| Partial failure | [D9](./ddia/9-distributed-trouble/) |
| Quorum | [D6](./ddia/6-replication/), [D10](./ddia/10-consistency-and-consensus/) |
| Relational model | [D3](./ddia/3-data-models/), [H6](./hard-parts/6-operational-data/) |
| Replication (storage) | [D6](./ddia/6-replication/) |
| Replicated cache | [H10](./hard-parts/10-distributed-data-access/) |
| Reuse (library / service / sidecar) | [H8](./hard-parts/8-reuse-patterns/) |
| Saga | [H12](./hard-parts/12-transactional-sagas/) |
| Serializability | [D8](./ddia/8-transactions/) |
| Service mesh / sidecar | [H8](./hard-parts/8-reuse-patterns/) |
| Sharding | [D7](./ddia/7-sharding/) |
| Stamp coupling | [H13](./hard-parts/13-contracts/) |
| Stream processing | [D12](./ddia/12-stream-processing/) |
| Tactical forking | [H4](./hard-parts/4-decomposition/) |
| Two-phase commit | [D8](./ddia/8-transactions/) (not a saga; see [H12](./hard-parts/12-transactional-sagas/)) |
| Vector index | [D4](./ddia/4-storage-and-retrieval/) |
| Workflow state | [H11](./hard-parts/11-distributed-workflows/) |
| Write skew | [D8](./ddia/8-transactions/) |

---

## Suggested paths (not a merge)

**Path 1 — "I need the data-systems spine."**  
DDIA 1 → 14 in order. Peek at Hard Parts 6 and 9 when DDIA 3 and 8 start
talking about "just put it in Postgres."

**Path 2 — "We are splitting a monolith and the database is the knot."**  
Hard Parts 1 → 7 (pull apart), then 8 → 14 (put back). Peek at DDIA 8
when HP 9/12 say "transaction," and at DDIA 5 when HP 13 says "contract."

**Path 3 — "Same word, both jobs."**  
Use the cross-book map. Read the DDIA folder, then the Hard Parts folder,
without copying paragraphs from one into the other.

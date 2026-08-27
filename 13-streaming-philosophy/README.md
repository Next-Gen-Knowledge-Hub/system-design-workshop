# 13. A philosophy of streaming systems

Companion notes for **Chapter 13** of *Designing Data-Intensive Applications*
(2nd ed.). The first edition folded this into “the future of data
systems.” The second edition makes it its own argument: **you will not
get one database that does everything well, so compose specialized
tools with dataflow**, and still aim for correctness.

## What this chapter is for

By now you have OLTP stores, warehouses, search, caches, stream jobs,
batch jobs. The naive integration is a web of dual writes. This chapter
is the alternative architecture: **one (or few) systems of record,
everything else derived**, asynchronously, replayably.

It is also the answer to “we cannot 2PC across Elasticsearch, Redis,
and Postgres” (ch. 8): **don’t.** Use identifiers, idempotency, and
asynchronous constraint checks — and be honest about what can be stale.

## Data integration

Every product at scale is several stores. The integration problem is
keeping them from drifting into different “truths.”

### Combining specialized tools by deriving data

**Derive, don’t dual-write.** The order service writes Postgres (system
of record). CDC or explicit events feed:

- search index
- warehouse / lake
- cache
- ML features
- notification pipeline

Each derived system is a **function of the record**. If the function is
wrong, you fix it and **recompute** (batch, ch. 11) or **replay** (log,
ch. 12). That is why log-based brokers and lakes matter: they are the
*inputs* you can run again.

Unbundling only works if the inputs are durable and ordered enough to
rebuild. Without that, every consumer is a snowflake sync job.

### Batch and stream processing together

Stream for freshness; batch for correction, backfill, and heavy
recompute. The hard part is making them **the same logical function**
so the nightly job does not disagree with the stream (**lambda
architecture**’s curse: two codepaths, two bugs).

Common compromises:

- **Kappa-ish:** the log is rich enough that “batch” is just replay with
  more parallelism; one codepath.
- Stream is the live path; batch periodically rebuilds from the lake to
  **heal drift**, then you investigate why drift happened.
- Shared libraries / SQL definitions so stream and batch compile the
  same transform.

## Unbundling databases

A traditional RDBMS already contains: a log, a query engine, secondary
indexes, materialized views, replication. **Unbundling** means using
separate products for those jobs — Kafka as the log, Elastic as an
index, Redis as a cached view, Flink as the view maintainer —
**composed around dataflow**.

### Composing data storage technologies

You are not throwing away databases. You are composing storage
technologies the way a database composes internal modules, with the
network in between. The cost is you now own consistency across them
(ch. 9). The gain is each piece can scale and evolve (ch. 5) on its
own schedule.

Rule of thumb: **writes enter at the system of record**; everything
else is a derived cache until proven otherwise.

### Designing applications around dataflow

A write is an event (or becomes one via CDC). APIs, UIs, and derived
stores are subscribers. Services stop “calling each other to update
state” in a mesh of RPCs and start **emitting facts** others apply.

You can push **derived state** all the way to a client (reactive UI,
local-first from ch. 6) so the phone updates when the view updates —
and still works offline if you designed for sync engines.

### Observing derived state

How does a client learn the view changed?

- Polling (simple, wasteful).
- Query-then-subscribe / watches (coordination services do this for
  cluster metadata; products do it with websockets, SSE, push).
- Push notifications when the materialized view updates.

The coordination-service watch is the ancestor; the product feature is
the same idea with user credentials and fan-out (ch. 2).

## Aiming for correctness

### The end-to-end argument for databases

Do not trust every hop to be perfect. Put **idempotency keys** on the
business operation so retries (ch. 9 timeouts) and double-clicks cannot
double-charge. The database’s atomicity is not enough if the user
generated two different request ids for one intent — or one id applied
twice across a replay.

End-to-end means: the *business* invariant is enforced where the
business knows enough, not only inside one storage engine’s
transaction.

### Enforcing constraints

Uniqueness and “sufficient balance” can be checked **asynchronously**
by a consumer that may **reject after the fact**. Product patterns:

- Wait for the check (higher latency, fewer apologies).
- **Optimistic accept + apologize** (email “we had to undo”; airlines
  overbook). Scalable; needs a clear user story and compensating
  transactions.

This is often more operable than distributed locking across services
or XA 2PC. It is not a license to corrupt money without a ledger —
ledgers still need a sharp consistency story (ch. 8 / 10) for the
record of truth.

### Timeliness and integrity

A derived view can be slightly **stale** (timeliness) as long as it is
not *wrong forever* (integrity). Linearizability (ch. 10) is for the
few facts that cannot be stale. Most of a social app is the other pile.

Say both numbers: “search may lag 5 s; it must not permanently miss a
delete.”

### Trust, but verify

Checksums, audit consumers that compare derived stores to the record,
lineage (which job produced this partition), row counts, and canaries.
Derived systems drift — bugs, dropped messages, schema mismatches
(ch. 5). Budget for **detecting** drift, not only for building pipes.

## How this shows up when you design something

A strong end-to-end design states:

1. The **system of record**.
2. **Derived** stores and how they are fed (CDC vs explicit events).
3. The **idempotency key** on the write API.
4. What is allowed to be **stale**, and by how long.
5. How you **rebuild** a derived store after a bug.
6. Which constraints are **sync** vs **async**, and how users see
   rejection.

That list beats a dozen product logos in an interview.

## Check yourself

1. Why is “write Postgres, write Elastic in the same request” not
   unbundling, and what goes wrong?
2. Give a constraint you would check asynchronously, and how the user
   sees a failure.
3. Timeliness vs integrity for a wallet balance vs a “people also
   bought” widget.
4. How does replay give you evolvability (ch. 2 / 5) that dual-write
   does not?
5. What is the lambda architecture’s main operational curse?
6. Why is Kafka-as-log + Elastic-as-index closer to an RDBMS’s internals
   than “we replaced the database with microservices”?
7. End-to-end idempotency: where does the key live, and what happens if
   two tabs submit twice with different keys?
8. Name one audit you would run weekly to catch derived-store drift.

Continue to [Doing the right thing](../14-doing-the-right-thing/).

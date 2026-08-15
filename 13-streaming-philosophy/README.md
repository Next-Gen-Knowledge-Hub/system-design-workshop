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
asynchronous constraint checks.

## Data integration

Every product at scale is several stores. The integration problem is
keeping them from drifting.

**Derive, don’t dual-write.** The order service writes Postgres (record).
CDC or explicit events feed:

- search index
- warehouse
- cache
- ML features
- notification pipeline

Each derived system is a **function of the record**. If the function
is wrong, you fix it and **recompute** (batch) or **replay** (log).
That is why log-based brokers and lakes matter: they are the *inputs*
you can run again.

**Batch and stream together:** stream for freshness, batch for
correction and backfill. The hard part is making them **the same
function** so the nightly job does not disagree with the stream
(lambda architecture’s curse). A common compromise: stream is the
live path; batch periodically rebuilds from the lake to heal drift
(or you trust the log enough to only replay — “Kappa”).

## Unbundling databases

A traditional RDBMS already contains: a log, a query engine, secondary
indexes, materialized views, replication. **Unbundling** means using
separate products for those jobs — Kafka as the log, Elastic as an
index, Redis as a view, Flink as the view maintainer — **composed
around dataflow**.

You are not throwing away databases. You are **composing storage
technologies** the way a database composes internal modules, with
the network in between. The cost is you now own consistency across
them (ch. 9). The gain is each piece can scale and evolve (ch. 5)
on its own.

**Applications as dataflow:** a write is an event; UI, API, and
derived stores are subscribers. You can push **derived state** all
the way to a client (reactive UI, local-first from ch. 6) so the
phone updates when the view updates, and still works offline if you
designed for it.

**Observing derived state:** watches, websockets, query-then-subscribe.
**State watch / query-then-subscribe** is the coordination-service version; here it
is the product version.

## Aiming for correctness

**End-to-end argument:** do not trust every hop to be perfect; put
**idempotency keys** on the business operation so retries (ch. 9
timeouts) cannot double-charge. The database’s atomicity is not
enough if the user clicked twice and you generated two ids.

**Enforcing constraints asynchronously:** uniqueness and “sufficient
balance” can be checked by a consumer that may **reject after the
fact**. Product patterns: wait for the check (higher latency), or
**apologize** (send the “sorry, we had to undo” email — airlines
overbook). This is often more scalable than distributed locking
across services.

**Timeliness vs integrity:** a derived view can be slightly stale
(timeliness) as long as it is not *wrong forever* (integrity).
Linearizability (ch. 10) is for the few facts that cannot be stale.
Most of a social app is the other pile.

**Trust, but verify:** checksums, audit consumers that compare derived
stores to the record, lineage. Derived systems drift. Budget for
detecting it.

## How this shows up when you design something

A strong end-to-end design:

1. Name the system of record.
2. Name derived stores and how they are fed (CDC vs explicit events).
3. Name the idempotency key on the write API.
4. Name what is allowed to be stale, and by how long.
5. Name how you rebuild a derived store after a bug.

That list is more impressive in an interview than a dozen product logos.

## Check yourself

1. Why is “write Postgres, write Elastic in the same request” not
 unbundling, and what goes wrong?
2. Give a constraint you would check asynchronously, and how the user
 sees a failure.
3. Timeliness vs integrity for a wallet balance vs a “people also
 bought” widget.
4. How does replay give you evolvability (ch. 2 / 5) that dual-write
 does not?

Continue to [Doing the right thing](../14-doing-the-right-thing/).

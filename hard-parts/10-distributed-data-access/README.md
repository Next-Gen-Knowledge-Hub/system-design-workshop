# 10. Distributed data access

These notes teach how another service reads data it does not own, from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## The join moved; it did not disappear

Ownership decides who writes. **Ownership** means one service is the system of record: it alone holds the write credentials for those rows. Once that is true, the SQL join is illegal. Service B needs a fact service A owns, and B must not read A’s tables. Every option below is a least-bad replacement for a join the engine used to do for free. You pick the pain you can operate.

You did not remove the join. You picked where it runs and how wrong it is allowed to be.

```
  Service B needs a fact Service A owns.

      A (writer)                         B (reader)
      +-----------+                      +-----------+
      │  system   │                      │  must not │
      │  of record│                      │  JOIN A's │
      │           │                      │  tables   │
      +-----------+                      +-----------+
            |                                  ^
            |    four least-bad joins          |
            |                                  |
            +---- 1. call A on the read path --+
            |       fresh · coupled · cascade  |
            |                                  |
            +---- 2. replicate some columns -->+
            |       local join · stale cols    |
            |                                  |
            +---- 3. replicated cache -------->+
            |       hot keys · invalidation    |
            |                                  |
            +---- 4. data-domain service ----->+
                    the join becomes a product
```

“Just call the other service” is a pattern with a cost, not a default. “Just replicate” is also a pattern. It is not the same thing as a database product copying its own log to a follower. A follower of A’s database is still A’s quantum: same product, same failover story, usually the whole dataset. A **quantum** is the unit you deploy and restore together, code plus the data it writes. A column copied into B’s database is a cross-quantum contract. A replicated cache is not multi-leader replication. Multi-leader means two copies both accept writes. A cache and a column copy must not.

Freshness inside one cluster — linearizability, a read guaranteed to see the latest committed write — is that cluster’s consistency. A stale *application* copy is an ownership and access choice. Same word, “consistency,” different job.

`HTTP GET` in a loop makes the neighbor’s latency and outage into your objective. Copying entire tables into every service “for speed,” with no invalidation story, spends a year on ghosts. Reads have an architecture. It is not `SELECT … JOIN`.

## Interservice communication

At read time, B asks A — a remote call, HTTP, maybe a query language aimed at A. A answers from the system of record. B joins in memory.

B cannot see A’s tables. The fact changes often enough that a copy would lie during the user’s session. The volume of the lookup is modest, or A is already on the request. Give B a read API with a tight contract: the fields B needs, not A’s internal row. Timeouts, retries that are safe for reads, and a bulk endpoint if the alternative is one call per item. A circuit breaker needs a defined degraded interface: empty, a cached value, or an error. Pick which.

Chatty entity getters — `GET /customers/1`, then `/2`, and so on — from a list endpoint reimplement a nested-loop join on the network. That is the N+1 you see in traces after a monolith split. The review demand is a bulk contract: one call that accepts the set of keys, or a copy so the list does not fan out. Treating A’s downtime as “unexpected” denies the choice you made. You chose runtime coupling. A’s error budget is now B’s. This is **dynamic coupling**: B cannot serve the screen if A is down.

This pattern maximizes freshness and ownership clarity. A still has the only truth. It is the least-bad join when the user would notice staleness of seconds, when the other service is on the path anyway, and when fan-out is bounded — one key or a few, not a table scan.

It is a bad replacement when B’s page is a report over thousands of A’s rows. That was a SQL join for a reason. You want a copy, a cache of an aggregation, or a data domain that *is* the report. A call for a warehouse-shaped screen will time out.

One user action that fans out to five owners is no longer “a join.” It is a workflow, or a backend-for-frontend that is becoming a data domain. Count the hops against a budget. Each synchronous hop is an objective you inherited. An API gateway that joins everything, because teams refused the four patterns, is the monolith again. The gateway is now the only place that knows how the screen is built.

## Column schema replication

Physically copy some columns — not necessarily whole rows — from A’s store into B’s store. B joins locally again. A remains the writer of the source. The copy is derived.

B joins A’s fact on almost every query: names, status, currency. Calling A would dominate latency and error rates. A delay of seconds to minutes is acceptable. Replicate a narrow, stable projection: the columns B is allowed to see, keyed for B’s queries. Populate it by events, by change data capture, or by a constrained batch. The plumbing may look like a stream of changes. The decision “B may store a copy of these columns” is the access pattern. A owns schema changes of the source. B owns the discipline of not treating the copy as writable.

Copying `SELECT *` so B can “do whatever” exports an entity-relationship diagram. Every `ALTER` on A breaks B. That is stamp coupling with extra disk: B depends on the exact shape of A’s row. B writing the copied columns is joint ownership, not access. Two writers, one fact.

Column replication is the least-bad join when the read path is hot and local, the projected columns rarely change shape, staleness has a number (an objective such as “name lag under 30 seconds”), and you can rebuild B’s copy from A. Backfill is a requirement, not a hope.

It is a bad replacement when the copied attributes *are* the life cycle B is trying to run. You needed to delegate the write to A, not to copy the column. It is also a bad replacement when A’s schema is a weekly moving target. You will spend the year migrating a shadow table. A column copy of a field the user edits in the other service *this minute* is a fight with the copy.

Do not confuse this with A’s read replica. A replica is still A: same product, same failover, usually the whole dataset. When that copy is wrong, A’s operators page. Column replication is a contractual subset living in B’s quantum: different operators, a different backup, a different lie. When B’s copy is wrong, B’s operators page, and they repair from A. “We replicated the database” in a service design has to say whose replica. If it is A’s follower, B still depends on A’s quantum. That is often fine for reporting. It is dishonest as independence.

Two copies and a bug: whose number is right? A is right. B’s copy is a cache with a schema. Measure divergence. Never repair A from B. Two-way sync of columns invents multi-leader replication across services: both sides accept writes, and conflicts have no single owner. Delete the reverse pipe.

## Replicated caching

B, or a fleet of B, keeps a cache of A’s facts: in-process, Redis, memcached, or a content-delivery network for public bits. Unlike a column copy, the cache is not the query planner’s table. It is a performance layer with a time-to-live or with explicit invalidation.

The same keys are read constantly. A would melt, or the hop would dominate. Slightly stale is acceptable for this view: feature flags, avatars, a product title, session-shaped documents. Cache with a stated freshness. Prefer invalidation events from A, or versioned keys, over guessing a time-to-live for data where correctness matters. Stamp the cache entry with the owner’s version so a resurrected old value is detectable.

“Redis, therefore architecture” has no expiry policy, no stampede control, no owner, and writes into the cache as if it were a database. Caching a join result that includes B’s own mutable rows, without a key that changes when *either* side changes, serves a Frankenstein row: half of it moved, the cache key did not.

| | Column copy in B’s database | Replicated cache |
|---|---|---|
| Query | Real joins, indexes, transactions with B’s rows | Get by key; joins in the application |
| Durability | Survives B’s restart | Often ephemeral |
| Staleness | Pipeline lag | Time-to-live or invalidation lag |
| Schema | Explicit columns (that coupling is the cost) | Opaque blob or a small struct |
| Fit | Many query shapes on the copy | Hot keys, simple lookup |

A cache is the least-bad join when the access is a point lookup of hot keys, not “give me everyone in region West.” A scan of a cache is a cry for a table (a column copy) or for a data domain.

Invalidation never quite works: a missed event, a time-to-live that is too long, clock tricks. Bound the lie. A maximum time-to-live, a version check, and a path that falls back to A on a miss or when the value must be right. If you cannot fall back, you needed a column copy you can rebuild, or you needed to call A.

Cache-aside from B, without A knowing, means A updates and B serves ghosts until expiry. For some screens that is fine. For “is this seat free?” it is a bug with a logo. A cache used to allocate the last item will oversell. That is a write-time invariant, not a read trick. The owner has to perform the allocation.

A cache of A inside A’s quantum — A’s Redis in front of A’s database — is A’s performance. It is not a distributed access pattern. This lesson starts when **B** holds the bytes.

A cache that is the only copy of a fact has lost the system of record. That is not access. That is a new owner by accident.

## The data-domain pattern

Create or designate a service whose product is the join: a data domain that owns a combined view, or that orchestrates the read so every consumer does not invent the first three patterns alone.

Many services need the same combination: ticket plus assignee plus a customer snippet, or order plus payment state plus shipment. Each team copies the same three calls or the same columns. The join *is* a business concept: a dispatch board, an account snapshot.

A read-oriented domain has a contract. It knows how fresh each part is, how to rebuild, and whom to page. The writers remain the single owners of the source facts. This domain may store a **serving view**. That view is derived data. It must not become a second writer of the source facts.

A “god read service” that becomes the only way to get any field, including fields a consumer should have called A for, hides the simple patterns inside one bottleneck. A domain that writes back into A and B “to keep them aligned” is a hidden orchestrator of writes, not an access pattern. Write orchestration belongs with ownership: delegate to the owner, or run an explicit workflow that compensates. It does not belong in a read model that quietly mutates sources.

This is the least-bad join when the combination is reused, when freshness rules are non-trivial (a mix of live and stale parts), when you want one place to apply authorization on the combined picture, and when the user journey is cheaper as one call to the view than as a mesh of peers. Search, recommendations, and a board interface are usually this pattern, or a projection owned the same way, not twenty calls.

It is a bad replacement when only one consumer needs the join. That consumer *is* the join — a backend-for-frontend, or a direct call to A — and an extra service is granularity theater. You staffed a service instead of a query. A one-off admin join is the same mismatch.

Data domains also exist on the write side: groups of tables with one life cycle and one owner. A serving view is not that split. Writes still have single owners. The new domain owns a projection.

## Which replacement to pick

There is no ranking from good to bad. There is a matching problem. Ask in order:

1. Is staleness forbidden on this path, and is fan-out tiny? Call A.
2. Is this a hot point-lookup with a bounded lie? Use a replicated cache, and fall back to a call on a miss.
3. Does B query the foreign fact in many shapes, locally? Replicate a narrow column projection.
4. Do many consumers share this combination and this mix of freshness? Build a data domain, a serving view.
5. Is the join actually a write-time invariant? You are in the wrong problem. Keep the tables in one quantum, or give the write to a single owner and delegate. A read trick will not hold an invariant.

A review that picks a pattern by fashion — “events,” “Redis,” “a graph of calls” — has not named the lie. Walk the questions. Write the lie you accepted: 0 milliseconds, 30 seconds, a five-minute time-to-live, nightly. If you cannot name the lie, you are still imagining a SQL join.

Using all four patterns for the same fact “for resilience” produces four lies and no owner for the screen. Pick one primary read path. Extra layers need a reason. A cache in front of a call is normal. A cache plus a column copy plus a domain plus a call is a maze.

A team that wants Redis *and* a shadow table *and* a live call for the same customer name deletes down to one primary. The name on a hot list, slightly stale, can be a cache with fallback. The name joined in many local query shapes can be a narrow column copy with a lag objective. The name that must be current on this path, for one or a few keys, is a call. It is not all three.

Sharp mismatches, collected:

- A call for a warehouse-shaped screen times out.
- A column copy of a field the user edits in the other service this minute fights the copy.
- A cache for “allocate the last item” oversells. That is a write invariant.
- A data domain for a one-off admin join staffs a service instead of a query.

## Recap

After a single owner holds the tables, a reader replaces the join in one of four ways. Call the owner when the value must be fresh and the fan-out is small, and pay with that owner’s outages. Copy a narrow, rebuildable projection when local queries dominate and a measured lag is acceptable; the owner stays the truth, and the reader does not write the copy. Cache hot keys with a bound on the lie and a fallback, and do not allocate scarce inventory from the cache. Stand up a read-oriented domain when many consumers share one combination and one freshness mix. One primary path per fact. A write-time invariant is not a read pattern.

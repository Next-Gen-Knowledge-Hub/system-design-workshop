# 10. Distributed data access

Companion notes for **Chapter 10** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021). Chapter 9 decided
**who writes**. This chapter decides **how someone else reads** once
the SQL join is illegal. Every option is a least-bad replacement for
a join you used to get for free. Pick the pain you can operate.

Skip it and you will `HTTP GET` in a loop until latency and a
neighbor’s outage become your SLO, *or* you will copy entire tables
into every service “for speed” and spend a year on invalidation.
Reads have an architecture. It is not `SELECT … JOIN`.

## See also (do not merge)

**Copies inside one database product** — leaders, followers,
quorums, lag, failover — are
[DDIA ch. 6](../../ddia/6-replication/). That chapter answers
“how does *this store* keep replicas honest?” This chapter
answers “**how does another service read my data?**” A follower
of *your* Postgres is still *your* quantum. A column copied
into *their* database is a **cross-quantum** contract. Do not
call a follower “the data access pattern,” and do not call a
replicated cache “multi-leader replication.”

Freshness of a *single cluster’s* reads (linearizability) is
[DDIA ch. 10](../../ddia/10-consistency-and-consensus/). Stale
*application* copies are an ownership + access choice
([ch. 9](../9-data-ownership/) plus this folder). Same English
word “consistency,” different job.

## The mental model

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

The sentence to keep: **you did not remove the join; you picked
where it runs and how wrong it is allowed to be.**

Two consequences. First, “just call the other service” is a
pattern with a cost, not a default. Second, “just replicate”
is also a pattern — and it is **not** DDIA replication unless
the copy is still inside A’s store under A’s failover story.

## Interservice communication

B, at read time, asks A (RPC, HTTP, maybe GraphQL to A). A
answers from the system of record. B joins in memory.

**Problem** — B cannot see A’s tables. The fact changes often
enough that a copy would lie during the user’s session. Volume
of the lookup is modest, or A is already in the request.
**Solution** — A **read API** with a tight contract: the fields
B needs, not A’s internal row. Timeouts, retries that are safe
for reads, a bulk endpoint if you would otherwise N+1. Circuit
breaker with a **defined** degraded UI (empty, cached, error).
**Failure mode** — Chatty entity getters (`GET /customers/1`,
then `/2`, …) from a list endpoint. You reimplemented a nested
loop join on the network. Also: treating A’s downtime as
“unexpected.” You chose runtime coupling; A’s error budget is
now B’s.

This pattern maximizes **freshness** and **ownership clarity**
(A still has the only truth). It maximizes **dynamic coupling**
([ch. 2](../2-coupling/)): B cannot serve the screen if A is
sad. It is the least-bad join when:

- the user would notice staleness of seconds,
- the other service is on the path anyway,
- fan-out is bounded (one or a few keys, not a table scan).

It is a bad join replacement when B’s page is a **report** over
thousands of A’s rows. That was a SQL join for a reason; you
want a copy, a cache of an aggregation, or a data domain that
*is* the report.

**Problem** — One user action fans out to five owners.
**Solution** — That is no longer “a join.” It is a workflow
or a BFF that is becoming a
[data domain](#data-domain-pattern). Count hops against the
budget from [ch. 7](../7-service-granularity/).
**Failure mode** — An API gateway that joins everything
because teams refused these four patterns. The gateway is now
the monolith.

## Column schema replication

Physically **copy some columns** (not necessarily whole rows)
from A’s store into B’s store. B joins locally again. A remains
the writer of the *source*; the copy is derived.

**Problem** — B joins A’s fact on **almost every query** (names,
status, currency). Calling A would dominate latency and error
rates. A delay of seconds to minutes is acceptable.
**Solution** — Replicate a **narrow, stable projection**: the
columns B is allowed to see, keyed for B’s queries. Populate
via events, CDC, or a constrained batch — the *plumbing* may
look like [DDIA ch. 12](../../ddia/12-stream-processing/); the
*decision* “B may store a copy of these columns” is this
chapter. A owns schema change of the source; B owns not
treating the copy as writable.
**Failure mode** — Copying `SELECT *` so B can “do whatever.”
You exported an ER diagram. Every `ALTER` on A breaks B.
That is stamp coupling with extra disk
([ch. 13](../13-contracts/)). Also: B **writes** the copied
columns. That is [joint ownership](../9-data-ownership/), not
access.

Column replication is the least-bad join when:

- read path is **hot and local**,
- the projected columns **rarely change shape**,
- staleness has a number (SLO: “name lag < 30s”),
- you can rebuild B’s copy from A (backfill is a requirement,
  not a hope).

It is a bad replacement when the copied attributes **are** the
life cycle B is trying to run (you needed to delegate writes,
not copy). It is also a bad replacement when A’s schema is a
weekly moving target — you will spend the year on migrations
of a shadow table.

Do not confuse this with **A’s read replica**. A replica is
still A: same product, same failover, usually the whole
dataset. Column replication is a **contractual subset** living
in B’s quantum. Different operators, different backup, different
lie.

**Problem** — Two copies, a bug, whose number is right?
**Solution** — A is right. B’s copy is a cache with a schema.
Measure divergence. Never “repair” A from B.
**Failure mode** — Two-way sync of columns. You invented
multi-leader across services. Stop. Read DDIA 6 for why that
hurts, then delete the reverse pipe.

## Replicated caching

B (or a fleet) keeps a **cache** of A’s facts: in-process,
Redis, memcached, a CDN for public bits. Unlike column
replication, the cache is **not** the query planner’s table;
it is a performance layer with TTL or explicit invalidation.

**Problem** — The same keys are read constantly; A would melt
or the hop would dominate; slightly stale is OK for this
view (feature flags, avatars, product title, session-ish
documents).
**Solution** — Cache with a **stated freshness**. Prefer
**invalidation events** from A (or versioned keys) over
guessing TTLs for correctness-sensitive data. Stamp the
cache with the owner’s version so you can detect resurrection
of old values.
**Failure mode** — “Redis, therefore architecture.” No TTL
policy, no stampede control, no owner, writes to the cache
as if it were a database. Also: caching a **join result**
that includes B’s own mutable rows without a key that
changes when *either* side changes. You will serve a
Frankenstein row.

Compared with column replication:

| | Column copy in B’s DB | Replicated cache |
|---|---|---|
| Query | Real joins, indexes, txns with B’s rows | Get by key; joins in app |
| Durability | Survives B’s restart | Often ephemeral |
| Staleness | Pipeline lag | TTL / invalidation lag |
| Schema | Explicit columns (coupling) | Opaque blob or small struct |
| Fit | Many query shapes on the copy | Hot keys, simple lookup |

Cache is the least-bad join when the access is **point-get of
hot keys**, not “give me everyone in region West.” A scan of
a cache is a cry for a table (column copy) or a data domain.

**Problem** — Invalidation never quite works (missed event,
TTL too long, clock tricks).
**Solution** — Bound the lie: max TTL, version checks, and
a path to **fall back to A** on miss or on “must be right.”
If you cannot fall back, you needed a column copy you can
rebuild, or you needed to call A.
**Failure mode** — Cache-aside from B without A knowing.
A updates; B serves ghosts until TTL. For some screens that
is fine. For “is this seat free?” it is a bug with a logo.

A cache of A inside A’s quantum (A’s Redis in front of A’s
DB) is **A’s performance**. It is not a distributed access
pattern. This chapter starts when **B** holds the bytes.

## Data domain pattern

Create (or designate) a service whose **product is the join**:
a new data domain that owns a combined view, or that
orchestrates the read so every consumer does not invent
pattern 1–3 alone.

**Problem** — Many services need the same combination (ticket
+ assignee + customer snippet; order + payment state +
shipment). Each team copies the same three calls or the same
columns. The join *is* a business concept (“dispatch board,”
“account snapshot”).
**Solution** — A **read-oriented domain** with a contract:
it knows how fresh each part is, how to rebuild, whom to
page. Writers remain the owners from ch. 9. This domain is
allowed to store a **serving view** (that view is derived
data; do not let it become a second writer of source facts).
**Failure mode** — A “God read service” that becomes the
only way to get *any* field, including ones a consumer
should have called A for. Or a domain that **writes** back
into A and B “to keep them aligned” — that is a hidden
orchestrator of writes ([ch. 9](../9-data-ownership/) /
[ch. 11](../11-distributed-workflows/)), not access.

This is the least-bad join when:

- the combination is **reused**,
- freshness rules are **non-trivial** (mix of live and stale),
- you want **one** place to apply authorization on the
  combined picture,
- hop-count for the user journey is better as **one** call
  to the view than as a mesh of peers.

It is a bad replacement when only **one** consumer needs the
join — then that consumer *is* the join (BFF or pattern 1)
and an extra service is granularity theater
([ch. 7](../7-service-granularity/)).

The data domain pattern is how [ch. 6](../6-operational-data/)
“domains” re-enter on the **read** side. Do not confuse a
serving view with splitting the write database. Writes still
have single owners. The new domain owns a **projection**.

## When each is the least-bad join replacement

There is no ranking from “good” to “bad.” There is a matching
problem.

```
  Need the join. Ask, in order:

  1. Is staleness forbidden on this path, and is fan-out tiny?
        -> interservice call
  2. Is this a hot point-get with a bounded lie?
        -> replicated cache (fallback to 1 on miss)
  3. Does B query the foreign fact in many shapes, locally?
        -> column schema replication (narrow projection)
  4. Do many consumers share this combination / freshness mix?
        -> data domain (serving view)
  5. Is the join actually a write-time invariant?
        -> you are in the wrong chapter (go back to 7 / 9)
```

**Problem** — A review that picks a pattern by fashion
(“events,” “Redis,” “graph of calls”).
**Solution** — Walk the questions. Write the **lie you
accepted** (0 ms, 30 s, TTL 5 min, nightly). If you cannot
name the lie, you are still imagining a SQL join.
**Failure mode** — Using all four for the same fact
“for resilience.” You now have four lies and no owner for
the screen. Pick **one primary** read path; extra layers
need a reason (cache in front of a call is normal; cache
plus column copy plus a domain plus a call is a maze).

A few sharp mismatches:

- **Call** for a warehouse-shaped screen: you will time out.
- **Column copy** of a field the user edits in the other
  service *this minute*: you will fight the copy.
- **Cache** for “allocate the last item”: you will
  oversell. That is a write invariant, not a read trick.
- **Data domain** for a one-off admin join: you will staff
  a service instead of a query.

## How this shows up when you design something

- N+1 in the traces after a monolith split: you defaulted
  to pattern 1 without a bulk API or a copy.
- “We replicated the DB” in a service design: ask *whose*
  replica. If it is A’s follower, B still depends on A’s
  quantum — often fine for reporting, dishonest as
  independence.
- Search, recommendations, or a board UI: usually a data
  domain or a projection, not twenty calls.
- A cache that is the only copy of a fact: you lost the
  system of record. That is not access; that is a new
  owner by accident.

## Check yourself

1. In one sentence, DDIA 6 vs this chapter: replica of a
   store vs another service’s read. Give a sentence you
   would reject for merging them.
2. Interservice reads: what makes N+1 the failure mode, and
   what bulk contract would you demand in review?
3. When is calling A on every page *better* than a column
   copy? Name freshness and fan-out.
4. Column schema replication: which columns do you refuse
   to copy, and what ownership bug appears if B writes
   them?
5. Why is A’s read replica not “column replication into B”?
   Who pages when the copy is wrong?
6. Replicated cache vs column copy: pick a screen for each.
   What query shape kills the cache pattern?
7. Invalidation missed an event. What bound (TTL, version,
   fallback) did your design require, and what product bug
   appears if it required none?
8. Data domain as join replacement: when is it granularity
   theater, and when is the combination itself a product?
9. A team wants Redis *and* a shadow table *and* a live
   call for the same customer name. What do you delete,
   and which pattern stays?
10. “This join is a write-time invariant.” Which chapter
    do you open instead of this one, and which technique
    (ownership or granularity) is the real fix?

Continue to [Distributed workflows](../11-distributed-workflows/).

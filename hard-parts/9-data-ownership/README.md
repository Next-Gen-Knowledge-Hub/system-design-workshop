# 9. Data ownership and distributed transactions

Companion notes for **Chapter 9** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021). After you split
operational data ([ch. 6](../6-operational-data/)) and size services
([ch. 7](../7-service-granularity/)), someone still has to be allowed
to `UPDATE` the row. This chapter is **who may write**, and what you
do when a business action now touches **two owners**. It is not a
lesson in isolation levels.

Skip it and every service will write every table “just this once,”
or you will reach for a distributed transaction coordinator because
the word “transaction” appeared in a meeting. You will get neither
clear ownership nor a store that can abort cleanly.

## See also (do not merge)

**Who may write across services** is this folder.
**What isolation means inside one database** is
[DDIA ch. 8](../../ddia/8-transactions/). Serializability, write
skew, SSI, and 2PC as a *commit protocol* live there. Do not
import “just set SERIALIZABLE” into a design that already
crosses two data domains — that knob does not span HTTP.

**Making replicas agree so they look like one copy** is
[DDIA ch. 10](../../ddia/10-consistency-and-consensus/)
(linearizability, Raft, etcd). That is not “eventual
consistency” as this chapter uses it. Here, eventual means an
**application protocol** between owners: background job,
request-path orchestration, or events. Consensus is how a
*cluster* elects a leader. Do not paste quorum diagrams into
a saga review.

Replica *lag* inside one product is
[DDIA ch. 6](../../ddia/6-replication/). How another *service*
is allowed to see your facts is
[ch. 10](../10-distributed-data-access/). Keep those three
apart: cluster copies, app-level catch-up, read paths.

## The mental model

```
  a row (or table) in operational data

       who is allowed to WRITE?
              |
              +-- SINGLE owner ---- others read (see ch. 10)
              |
              +-- COMMON owner ---- reference data; junk-drawer risk
              |
              +-- JOINT owners ---- still two writers
                      |
                      +-- table split
                      +-- new data domain
                      +-- delegate writes
                      +-- consolidate services

  a business action that needs two owners
              |
              you lost BEGIN/COMMIT across the seam
              |
              +-- 2PC / XA           --> DDIA 8, not your architecture
              +-- background sync    --> lag, simple
              +-- orchestrated req.  --> user waits, compensate
              +-- event-based        --> loose, “has it happened?”
```

The sentence to keep: **ownership is a write permission; consistency
across owners is an application protocol, not a database flag.**

Two consequences. First, “we share the table but we’re careful”
is joint ownership, which is an unmade decision. Second, “eventual
consistency” in a design doc that also says “Raft” is a merged
chapter; send the author to the See also links.

The book’s ticketing/field-ops case is only a label for “two
teams think they own the assignment columns.” Your version will
be inventory vs orders, profile vs identity, or wallet vs ledger.

## Assigning ownership

Every table (and eventually every column that can be updated)
needs a story. Three stories exist. Pick one on purpose.

### Single ownership

One service is the **system of record**. It is the only process
with write credentials. Everyone else asks, or reads a copy.

**Problem** — After a split, two ORMs still point at `tickets`.
**Solution** — One owner. Revoke the other writers. Reads go
through an API, a replica, a cache, or a column copy
([ch. 10](../10-distributed-data-access/)).
**Failure mode** — “Owner” in a wiki, write creds still in
three secrets managers. Ownership is the credential and the
migration repo, not the slide.

Single ownership is the default you want. It makes
[granularity](../7-service-granularity/) real: the quantum
includes the data. The cost is **every foreign fact is a seam**.
Pay it with ch. 10, not with a second writer “for convenience.”

### Common ownership

Some data is **not a domain’s heart** but everyone needs it:
country codes, holiday calendars, reason-code lists, maybe a
company-wide “service region” table. No team feels like the
product owner; every team feels like a reader who occasionally
inserts a row.

**Problem** — Reference data lives in the monolith leftover
schema because “it isn’t really ours.”
**Solution** — Give it an owner anyway: a small reference-data
quantum, or an explicit **common** store with a change process
(who may add a code, how it versions, how consumers hear).
Treat it as a product with a boring SLA.
**Failure mode** — Common becomes a **junk drawer**: audit
columns, feature flags, “misc.” Common is for *slowly changing
reference* facts, not for anything with a life cycle. The
moment a table has workflows, it needed single ownership.

Common ownership is an integrator pretending to be a kindness.
Audit it twice a year. If two domains fight over writes, it
was never common; it was joint.

### Joint ownership

Two (or more) services **both write** the same table or
overlapping columns. This is the usual leftover of a service
split that did not finish [ch. 6](../6-operational-data/)
assignment.

**Problem** — Dispatch updates `assignee`; the ticket service
updates `assignee`; last write wins; nobody can explain a
missing technician.
**Solution** — Do not live here. Joint is a **transitional
diagnosis**. Resolve it with one of the four techniques
below.
**Failure mode** — Row-level conventions (“we only touch our
columns”) without DB enforcement. Someone adds a trigger, a
batch job, or a one-off SQL and the convention dies. If the
database cannot tell the writers apart, you do not have a
design.

Joint ownership is also how **lost updates** return — not as
an isolation anomaly inside one txn
([DDIA ch. 8](../../ddia/8-transactions/)), but as two
applications stomping a field with no common snapshot. Same
pain, different layer. Do not “fix” it by turning up
isolation on a store that is not the only writer.

## Joint techniques

Four ways out. They are not equally polite.

### Table split

**Problem** — The table is a sandwich of two domains’ columns
(ticket header vs assignment/routing vs billing snapshot).
**Solution** — Split **tables** (or obvious column groups)
so each owner has a private relation, keyed by a shared id.
Writes no longer collide. Reads that need both go to ch. 10.
**Failure mode** — Split tables, keep a view that is
*writable*, or a trigger that copies both ways forever. You
have dual-write with extra DDL.

Split along **invariants**, not along “this column is a
string.” If `status` is part of both life cycles, splitting
the table without splitting the status machine just moves
the fight to a new name.

### Data domain

**Problem** — The contested data is a **third** concept
(assignment, wallet, entitlement) that neither existing
service should own.
**Solution** — Create a new data domain (and likely a
service) whose job is that concept. Both former writers
become clients. This is an extractor, not a rename.
**Failure mode** — A “domain” that is only a pass-through
`UPDATE` with no extra invariant. You added a hop and kept
the joint write in spirit. New domains need a reason to
exist beyond peacekeeping.

This technique shows up when [granularity](../7-service-granularity/)
said the *verb* was a product. Ownership must follow that
verb into its own store, or you only split the code.

### Delegate

**Problem** — Service B needs a mutation that belongs in
A’s table. Giving B a password would be joint ownership.
**Solution** — **A remains the only writer.** B asks A
(command, RPC, message A consumes). A enforces invariants
and commits locally.
**Failure mode** — Delegate in name, `UPDATE` in a “break
glass” job that runs from B’s pipeline. Also: a chatty
delegate on the user path that should have been a table
split or a merge.

Delegate is the technique that preserves a **single owner**
without pretending B never needed the side effect. It is
also how you get a workflow ([ch. 11](../11-distributed-workflows/)).
If almost every request is a delegate, you may have split
the wrong seam — consolidation is allowed.

### Service consolidation

**Problem** — The two services are in a permanent 2-phase
dance, share code, share data relationships, and the
disintegrators never paid off.
**Solution** — Merge them. One quantum, one database, one
transaction. This is [ch. 7](../7-service-granularity/)
integrators winning after a field test.
**Failure mode** — Consolidation as taboo. A year of sagas
for a `status` flag is not “event-driven maturity.” It is
an unfinished split.

Use consolidation when the **write-time invariant** is the
product and the user cannot be shown an in-between state.
Use it late enough that you learned, early enough that the
saga machinery has not become a religion.

## Distributed transactions (why 2PC is the other chapter)

When one business action must mutate two owners, people say
“distributed transaction.” Be precise.

**Two-phase commit** is a protocol: prepare, then commit or
abort, so several *resource managers* agree whether **this
atomic unit** happened. Blocking, in-doubt sessions, a
coordinator that must not forget. Heterogeneous XA (Postgres
+ Kafka + search in one txn) is operationally infamous.
That material — including when DB-internal 2PC is sane
inside **one product** — is
[DDIA ch. 8](../../ddia/8-transactions/). Read it there.

**This chapter’s job** is to refuse 2PC as the *shape of
your service architecture*. If every cross-owner action is
a distributed atomic commit, you have rebuilt one database
out of network calls, with worse failure modes
([DDIA ch. 9](../../ddia/9-distributed-trouble/) on partial
failure — mention only).

**Problem** — After the split, “open ticket and reserve
part” used to be one `COMMIT`.
**Solution** — Either **don’t split that invariant**
(consolidate / keep one domain) or accept **a window**
where the world is half-updated and design how you close
it (next section).
**Failure mode** — 2PC across two microservices’ databases
as the default. You paid for independent deploy and then
coupled their availability at prepare-time. You also did
not get isolation across the pair the way a single engine
would; you got atomic *commit*, which is not the same
gift.

If a vendor says “our mesh does distributed transactions,”
translate: either 2PC in a tuxedo, or a saga with
marketing. Name which. Sagas themselves are
[ch. 12](../12-transactional-sagas/); this chapter only
needs you to know **why you left 2PC** and **what three
application patterns replace the *feeling* of one commit**.

## Eventual consistency patterns

“Eventual” here means: **each owner commits locally**; the
*pair* of facts converges later by an application mechanism.
It is not replica repair inside Cassandra, and it is not
Raft.

All three patterns need **idempotency** and a story for
“the user asked, we timed out, we do not know.” Timeouts
are [DDIA ch. 9](../../ddia/9-distributed-trouble/); the
architectural choice of *how you sync* is here.

### Background synchronization

A job (cron, poller, nightly) reads A and updates B, or
diffs them and repairs.

**Problem** — B must roughly match A; a delay of minutes
to hours is allowed (reporting, search index, denormalized
display names, cache of region names).
**Solution** — Periodic reconcile, preferably from the
**system of record** outward. Make it restartable. Alert
on lag, not only on job failure.
**Failure mode** — Background sync for **money or
inventory** the user will act on in the same session.
Also: two-way sync (“whoever is newer”). You built
multi-leader without saying so
([DDIA ch. 6](../../ddia/6-replication/) is that monster;
do not reimplement it in Python).

Background is the cheapest pattern and the least honest
with the user. Fine for derived data. Insulting for
checkout.

### Orchestrated request-based

In the **user’s request**, an orchestrator (or the
originating service) calls owner A, then owner B. If B
fails, it tries to undo A (compensate) or retries B
before returning.

**Problem** — The user cannot be told “we’ll settle
overnight.” They need a yes/no now, but you still refuse
2PC.
**Solution** — Request-path orchestration with explicit
**compensation** and a timeout budget. The orchestrator
owns the in-flight state for that request. This previews
[ch. 11](../11-distributed-workflows/) (who holds
workflow state) and [ch. 12](../12-transactional-sagas/).
**Failure mode** — Happy-path calls, no undo, hope. Or
compensation that is not idempotent (refund twice). Or
an orchestrator that is “just the API gateway” with no
durability — a crash after A committed and before B
started is now a ghost.

Request-based is **tighter coupling** than events: A and
B must be up together for the user action. You kept a
slice of the old transaction’s *timing* without its
atomic commit. Say that out loud so nobody thinks you
still have `ROLLBACK`.

### Event-based

Owner A commits locally and emits an event (outbox if
you are doing it honestly — the outbox *mechanics* sit
with [DDIA ch. 8](../../ddia/8-transactions/) /
[ch. 12](../../ddia/12-stream-processing/); the
*architectural choice* to sync this way sits here).
Owner B consumes and commits its own store.

**Problem** — You want owners to deploy and fail
independently; a lag of seconds is tolerable; many
consumers may care.
**Solution** — Events as the cross-owner protocol.
Document **what B may assume**, how duplicates are
ignored, and how you detect a stuck consumer.
**Failure mode** — Dual-write app→DB and app→bus
without an outbox (split brain on crash). Or treating
“eventual” as “we don’t measure.” Or using the event
payload as a remote `UPDATE` API so every schema
change breaks the fleet ([ch. 13](../13-contracts/)).

Event-based is not consensus. B can be behind; a
read-your-writes user may see A’s fact and not B’s.
If that user experience is forbidden, you wanted
request-based, a single owner, or a read pattern from
[ch. 10](../10-distributed-data-access/) that is honest
about freshness.

### Choosing among the three

| Pattern | User sees | Coupling | Typical fit |
|---|---|---|---|
| Background | Stale until the job | Lowest at runtime | Derived, reports, slow reference |
| Orchestrated request | Wait / error now | High on the path | Checkout-like, must answer |
| Event-based | “Accepted”; B catches up | Low runtime, contract coupling | Multiple consumers, independent deploy |

**Problem** — A diagram that says “eventually consistent”
with no box from this table.
**Solution** — Pick a row. Write the lag SLO and the
compensation (or the “we show stale”) sentence.
**Failure mode** — Mixing all three for the same pair of
tables “for resilience.” You cannot debug three protocols
on one invariant.

## How this shows up when you design something

- Two `DATABASE_URL`s, both with `INSERT` on `orders`:
  joint ownership. Pick a technique before a saga catalog.
- “We’ll add 2PC.” Send them to DDIA 8, then ask which
  *application* pattern they meant.
- “Raft will keep the services consistent.” That sentence
  merged DDIA 10 into this chapter. Raft does not assign
  table owners.
- A nightly job that repairs checkout: wrong pattern
  family. Promote to request or events, or merge services.

## Check yourself

1. Single vs common vs joint: give a production table for
   each. What goes wrong if you mis-label joint as common?
2. Why is a wiki owner plus three write credentials a
   failure mode of *single* ownership?
3. Table split vs new data domain: both create a boundary.
   When is the contested data *not* a sandwich of columns
   but a third product?
4. Delegate vs consolidate: what hop-count or invariant
   would make you merge instead of asking A to write?
5. State, in one sentence, why 2PC is the DDIA chapter.
   What do you still have to design if you refuse it?
6. Isolation (write skew) vs two services writing one
   column: same user-visible bug, different layer. How
   would you tell which you have?
7. Why is “eventual consistency” in this chapter not
   linearizability and not replica repair? Steal the See
   also lines, then give a counterfeit sentence you would
   reject in review.
8. Background vs request-based: pick a user action that
   must not use background, and a derived view that must
   not use request-based. Why?
9. Event-based sync: name two failure modes that are
   *application* (not Kafka configuration). Include one
   about dual-write.
10. A team proposes 2PC *and* events *and* a nightly
    reconcile for the same two tables. What do you make
    them delete first, and which ownership technique
    might remove the need?

Continue to [Distributed data access](../10-distributed-data-access/).

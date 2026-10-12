# 9. Data ownership and distributed transactions

These notes teach who may write, and what replaces one commit across owners, from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## Ownership is a write permission

After operational data is split and services are sized, someone still has to be allowed to `UPDATE` the row. This lesson is who may write, and what you do when a business action touches two owners. It is not a lesson in isolation levels.

An **architecture quantum** includes the data a service writes. **Ownership** is the write permission on a table or on the columns that can change: which process holds the credential and the migration repository. **Single ownership** means one service is the system of record. **Common ownership** means reference data with a boring change process. **Joint ownership** means two or more services both write the same table or overlapping columns. Joint ownership is an unmade decision. “We share the table but we’re careful” is joint ownership.

A business action that used to be one `BEGIN` / `COMMIT` now crosses owners. You lost that atomic commit. Consistency across owners is an application protocol, not a database flag. Each owner commits locally. The pair of facts converges later by a background job, by a request that calls both and can compensate, or by an event the second owner consumes. That is what **eventual consistency** means here.

Three neighboring ideas use the same English words and do different jobs.

Isolation inside one database — serializability, write skew, snapshot isolation — is a knob on that one store. It does not span an HTTP call. Turning the knob up does not assign an owner.

Making replicas of one cluster agree so they look like one copy is a consensus problem. A protocol such as Raft elects a leader so the cluster can present one value. That is not an application catching up between two services, and it does not assign table owners. A design sentence that says “eventual consistency” and “Raft” in the same breath has merged those jobs.

A follower replica lagging its leader, inside one database product, is that product’s copy falling behind. How another *service* is allowed to see your facts is a read pattern: a call, a column copy, a cache, or a serving view. Keep the three apart: cluster copies, application catch-up between owners, and read paths.

The book’s ticketing case is a label for “two teams think they own the assignment columns.” The same fight is inventory versus orders, profile versus identity, or wallet versus ledger.

```
  a row (or table) in operational data

       who is allowed to WRITE?
              |
              +-- SINGLE owner ---- others read by a defined pattern
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
              each side commits locally; the pair is not one COMMIT
              |
              +-- two-phase commit     couples availability; not the architecture
              +-- background sync      lag, simple, wrong for the live session
              +-- orchestrated request user waits, compensate on failure
              +-- event-based          loose; “has it happened yet?”
```

Every service writing every table “just this once” destroys ownership. Reaching for a distributed transaction coordinator because the word “transaction” appeared in a meeting gives you neither a clear owner nor a store that can abort the whole action cleanly.

## Single ownership

One service is the system of record. It is the only process with write credentials. Everyone else asks it to change the row, or reads a copy.

After a split, two object-relational maps still point at `tickets`. Revoke the other writers. Reads go through an API, a replica of the owner’s own database, a cache, or a column copy in the reader’s store. The owner remains the only truth.

“Owner” on a wiki, with write credentials still in three secret stores, is not ownership. Ownership is the credential and the migration repository.

Single ownership is the default. It makes a service boundary real, because the quantum includes the data. The cost is that every foreign fact is a seam. Pay for that seam with a read pattern. Do not pay for it with a second writer “for convenience.”

## Common ownership

Some data is not a domain’s heart, and everyone needs it: country codes, holiday calendars, reason-code lists, maybe a company-wide service-region table. No team feels like the product owner. Every team feels like a reader who occasionally inserts a row.

Reference data that lives in the leftover monolith schema because “it isn’t really ours” still needs an owner. Give it a small reference-data quantum, or an explicit common store with a change process: who may add a code, how the list versions, how consumers hear. Treat it as a product with a boring objective.

Common becomes a junk drawer when audit columns, feature flags, and “misc” land there. Common is for slowly changing reference facts, not for anything with a life cycle. The moment a table has workflows, it needed a single owner.

Common ownership is an integrator pretending to be a kindness. Audit it twice a year. If two domains fight over writes, it was never common. It was joint.

Mis-labeling joint as common is how a balance, an assignee, or a status machine ends up in the shared drawer. Last write wins, and the change process for country codes is now the change process for money.

## Joint ownership

Two or more services both write the same table or overlapping columns. This is the usual leftover of a service split that assigned code and never assigned tables.

Dispatch updates `assignee`. The ticket service updates `assignee`. Last write wins. Nobody can explain a missing technician.

Do not live here. Joint ownership is a transitional diagnosis. Resolve it with one of the four techniques below.

Row-level conventions — “we only touch our columns” — without the database enforcing the writers, die the day someone adds a trigger, a batch job, or a one-off SQL statement. If the database cannot tell the writers apart, you do not have a design.

Joint ownership is also how lost updates return. Inside one transaction, a lost update is an isolation anomaly: two transactions read the same row and each writes back, and one overwrites the other because they did not share a strict enough snapshot. Here, two applications stomp a field and there is no common snapshot at all. The user sees the same bug. The layer is different. Raising the isolation level on a store that is not the only writer does not fix the second writer.

Two connection strings that can both `INSERT` into `orders` are joint ownership. Pick a technique before you catalog sagas.

## Four ways out of joint ownership

They are not equally polite.

### Table split

The table is a sandwich of two domains’ columns: a ticket header, assignment and routing, a billing snapshot. Split tables, or split obvious column groups, so each owner has a private relation keyed by a shared id. Writes no longer collide. A read that needs both becomes an access pattern: call, copy, cache, or a serving view.

Split along invariants, not along “this column is a string.” If `status` is part of both life cycles, splitting the table without splitting the status machine moves the fight to a new name.

Split tables that keep a writable view, or a trigger that copies both ways forever, are dual-write with extra schema. You still have two writers.

### A new data domain

The contested data is a third concept — assignment, a wallet, an entitlement — that neither existing service should own. Create a new data domain, and likely a service, whose job is that concept. Both former writers become clients. This is an extractor, not a rename.

A “domain” that is only a pass-through `UPDATE`, with no extra invariant, adds a hop and keeps the joint write in spirit. A new domain needs a reason to exist beyond peacekeeping.

This is the technique that follows when a verb was actually a product: the routing algorithm, the wallet, the entitlement check. Ownership has to follow that verb into its own store. Splitting only the code leaves both sides writing the old table.

### Delegate

Service B needs a mutation that belongs in A’s table. Giving B a password would be joint ownership. A remains the only writer. B asks A, by a command, a call, or a message A consumes. A enforces the invariants and commits locally.

Delegate preserves a single owner without pretending B never needed the side effect. The sequence of asks is a workflow: someone has to hold the in-flight state if more than one owner is involved.

Delegate in name, with an `UPDATE` in a break-glass job that runs from B’s pipeline, is joint ownership again. A chatty delegate on the user path, where almost every request is “please write this for me,” means the seam was wrong. A table split or a merge is allowed.

### Service consolidation

The two services are in a permanent two-step dance. They share domain code, they share write-time relationships, and the reasons to split never paid off. Merge them. One quantum, one database, one transaction. The integrators won after a field test.

Consolidation treated as taboo produces a year of sagas for a `status` flag. That is not event-driven maturity. It is an unfinished split.

Use consolidation when the write-time invariant is the product and the user cannot be shown an in-between state. Use it late enough that you learned from the split, and early enough that the saga machinery has not become a religion.

## Why two-phase commit is not the architecture

When one business action must mutate two owners, people say “distributed transaction.” Be precise.

**Two-phase commit** is a protocol. A coordinator asks each resource manager to prepare, then tells them all to commit or all to abort, so they agree whether this atomic unit happened. It blocks. It leaves in-doubt sessions. The coordinator must not forget the outcome. Heterogeneous XA is operationally infamous. XA is the interface a coordinator uses to run two-phase commit across different products. The case people mean is one transaction spanning Postgres, Kafka, and a search index. Inside *one* database product, an internal two-phase commit can be a sane way for that product to commit its own resources. That is a property of the engine.

The architectural refusal is different. If every cross-owner action is a distributed atomic commit, you have rebuilt one database out of network calls, with worse failure modes. Networks lose responses. A caller can time out after the work succeeded and not know which. Partial failure is normal, not exceptional.

“Open ticket and reserve a part” used to be one `COMMIT`. Either do not split that invariant — consolidate, or keep one data domain — or accept a window where the world is half-updated and design how you close it.

Two-phase commit across two services’ databases, as the default, spends the independent deploy and then couples their availability at prepare time. You also do not get the isolation a single engine would give the pair. You get atomic *commit*, which is not the same gift.

A vendor who says “our mesh does distributed transactions” is offering one of two things: two-phase commit in a tuxedo, or a saga with marketing. Name which. A **saga** is the application protocol that replaces the feeling of one commit: each owner commits locally, and a failed later step runs a compensating action. The saga’s state machine, and who drives it, is a further design. This lesson only needs why you left two-phase commit, and which of the three patterns below replaces the feeling.

## Three ways the pair of facts converges

“Eventual” here means each owner commits locally and the pair converges later by an application mechanism. It is not replica repair inside a wide-column cluster, and it is not Raft electing a leader.

All three patterns need **idempotency**: doing the same request twice does not double the effect. All three need a story for “the user asked, we timed out, we do not know.” A timeout is not a clean failure. The response can be lost after the remote side committed. The architectural choice is *how you sync*. The uncertainty is a property of the network either way.

### Background synchronization

A job — a cron, a poller, a nightly run — reads A and updates B, or diffs them and repairs. B must roughly match A, and a delay of minutes to hours is allowed: reporting, a search index, denormalized display names, a cache of region names.

Reconcile periodically, preferably from the system of record outward. Make the job restartable. Alert on lag, not only on job failure.

Background sync for money or inventory the user will act on in the same session lies to that session. Two-way sync — “whoever is newer” — is multi-leader replication built in application code: two copies both accept writes and then try to reconcile conflicts. That conflict handling is a hard problem inside one database product. Reimplementing it as a Python job does not make it easier. Stop the reverse pipe.

Background is the cheapest pattern and the least honest with the user. It fits derived data. It is an insult at checkout. A nightly job that repairs checkout is the wrong family. Promote that pair to a request-path protocol or to events, or merge the services.

### Orchestrated request-based

In the user’s request, an orchestrator — or the originating service — calls owner A, then owner B. If B fails, it tries to undo A (compensate) or retries B before returning. The user cannot be told “we’ll settle overnight.” They need a yes or a no now, and you still refuse two-phase commit.

The orchestrator owns the in-flight state for that request, with an explicit compensation and a timeout budget. Holding workflow state, and the saga’s compensating steps, are the next designs. This pattern is the preview: the user waits, and undo is part of the protocol.

Happy-path calls with no undo are hope. Compensation that is not idempotent refunds twice. An orchestrator that is “just the API gateway,” with no durable state, loses the plot on a crash after A committed and before B started. That case is now a ghost: A’s side effect exists, and nobody remembers to finish or undo it.

Request-based coupling is tighter than events. A and B must be up together for the user action. You kept a slice of the old transaction’s *timing* without its atomic commit. Say that out loud so nobody thinks you still have `ROLLBACK`.

### Event-based

Owner A commits locally and emits an event. Owner B consumes it and commits its own store. You want the owners to deploy and fail independently. A lag of seconds is tolerable. Many consumers may care.

The honest write is an **outbox**: A inserts the business row and the outgoing event record in the same local transaction. A separate publisher reads that outbox and puts the event on the bus. The architectural choice to sync this way sits here. The crash window the outbox closes is mechanical: without it, the application writes the database and the bus in two calls, and a crash between them splits the brain. One side has the fact. The other never will, or the bus has an event for a row that rolled back.

Document what B may assume, how duplicates are ignored, and how you detect a stuck consumer. “Eventual” with no measurement is not a pattern.

Using the event payload as a remote `UPDATE` of B’s private schema means every change to A’s fields breaks the fleet. Callers are coupled to the stamp of the payload. Send the fact B is allowed to depend on, and let B own its tables.

Event-based sync is not consensus. B can be behind. A user who just wrote to A may read B and miss the fact. If that experience is forbidden, you wanted the request-based pattern, a single owner, or a read pattern that states its freshness instead of pretending the copy is current.

### Picking one

| Pattern | User sees | Coupling | Typical fit |
|---|---|---|---|
| Background | Stale until the job | Lowest at runtime | Derived data, reports, slow reference |
| Orchestrated request | Wait, or an error now | High on the path | Checkout-like; must answer |
| Event-based | “Accepted”; B catches up | Low at runtime, coupling through the contract | Several consumers, independent deploy |

A diagram that says “eventually consistent” with no row from this table has not picked a mechanism. Pick a row. Write the lag you allow, and write either the compensation or the sentence “we show stale.”

Mixing all three for the same pair of tables “for resilience” leaves three protocols on one invariant. You cannot debug that. A team that proposes two-phase commit *and* events *and* a nightly reconcile for the same two tables deletes down to one application pattern first. An ownership technique — table split, a new domain, delegate, or consolidate — may remove the need for any of them.

“Raft will keep the services consistent” does not survive review. Raft elects a leader inside a cluster. It does not decide who may `UPDATE` the row.

## Recap

One writer per table is the default: the credential, not the wiki. Common ownership is for slowly changing reference data, audited so it does not become a junk drawer. Joint ownership is transitional; leave it by splitting tables along invariants, extracting a real third domain, delegating the write to the single owner, or merging the services. Two-phase commit across service databases couples availability and is not the architecture. The pair of facts then converges by a background job, by a request-path orchestrator that can compensate, or by events with an outbox — one pattern per invariant, with idempotency and a timeout story.

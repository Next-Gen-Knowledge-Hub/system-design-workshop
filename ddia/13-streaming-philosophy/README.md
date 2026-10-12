# A philosophy of streaming systems

These notes teach the philosophy of streaming systems from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What this argument is

You will not get one database that does everything well. Compose specialized tools with dataflow, and still aim for correctness. An earlier telling of this material folded the argument into a chapter about the future of data systems. It stands on its own: the architecture, not a prediction.

A product at scale already has several stores. An OLTP database takes the order. A warehouse answers analytical questions. Search serves the product index. A cache serves the hot rows. Stream jobs and batch jobs fill those derived stores. The naive way to connect them is a web of dual writes: each request writes every store itself. Those stores then drift into different truths. The alternative is one system of record, or a few, and **everything else derived**, asynchronously, in a way you can replay.

This is also the practical answer to a distributed transaction across Elasticsearch, Redis, and Postgres. **Two-phase commit** (2PC), including XA between different database products, tries to make several stores commit or abort together. Do not use it across those three. Use identifiers, idempotency, and constraint checks that can finish after the request, and be honest about what is allowed to be stale.

## One record, many derived copies

### Derive, do not dual-write

An order service writes Postgres. Postgres is the system of record. From that commit, change data capture or an explicit event feeds the other copies:

- the search index
- the warehouse or the lake
- the cache
- machine-learning features
- the notification pipeline

Each of those is a **function of the record**. Search is "orders, indexed for lookup." The cache is "the hot order rows." Features are "what the model needs to know about this customer." If the function is wrong, you fix the function and run it again. A bounded recompute walks the stored input from start to end. A replay walks the log from an offset. Log-based brokers and lakes matter because they **are** those inputs. Without a durable, ordered input, every consumer is its own snowflake sync job, and a bug has nothing to rerun.

Writing Postgres and Elasticsearch in the same request is not this architecture. It is a dual write wearing an extra product logo. The request can commit the order and fail the index update. Search then permanently misses an order that the database has. Unbundling only works when the input is durable and ordered enough to rebuild. The log or the lake is that input. A second write inside the request is not.

### The same function, live and overnight

Use the stream for freshness. Use the batch job for correction, for backfill, and for a heavy recompute that should not run on the live path. The hard part is making them **the same logical function**. If the nightly job and the stream job implement the rule differently, they disagree, and the disagreement looks like data.

That is the curse of a **lambda architecture**: two codepaths, two bugs. You fix the stream, forget the batch, and tomorrow's rebuild undoes the fix. Or the reverse.

Three compromises show up in practice.

**Kappa-style.** The log is rich enough that "batch" is a replay of that same log with more parallelism. One codepath. The overnight job is the live job, started from an earlier offset and run harder.

**Heal from the lake.** The stream is the live path. A batch job periodically rebuilds the derived store from the lake. Where they differ, the rebuild wins, and you then investigate why they drifted. You still have two paths, but the batch path is the correction, not a second invention of the rule.

**One definition, two runtimes.** A shared library, or one SQL statement, compiles into both the stream job and the batch job. The rule lives in one place. The engines differ.

## A database, taken apart and wired back together

A traditional relational database already contains a log, a query engine, secondary indexes, materialized views, and replication. Those are modules inside one process. **Unbundling** uses a separate product for each job and composes them with dataflow. Kafka holds the log. Elasticsearch holds an index. Redis holds a cached view. Flink maintains the view. The network sits between the modules that used to share an address space.

You are not throwing the database away. You are composing storage technologies the way a database composes its own modules. The cost is that consistency across them is now yours. The database used to own it inside one transaction. The gain is that each piece can scale, and can change, on its own schedule. You can rebuild the index without resizing the OLTP box. You can replace the cache without rewriting the log.

The rule of thumb: **writes enter at the system of record**. Everything else is a derived cache until you have a reason to promote it.

Kafka as the log plus Elasticsearch as the index is close to what a relational engine already is internally: an append-only history, and a secondary structure maintained from that history. That is a different claim from "we replaced the database with microservices." A mesh of services that call each other to poke at state has no single log to replay. An unbundled database still has one.

### Applications that emit facts

A write is an event, or it becomes one through change data capture. APIs, user interfaces, and derived stores are subscribers. They apply the fact. They do not form a mesh of remote calls in which each service tells the next one to update its private copy.

Derived state can be pushed all the way to the client. A **reactive UI** updates when the view updates. **Local-first** software, built on a sync engine, goes further: the phone keeps its own copy, applies updates when they arrive, and still works offline because the sync was designed for that.

### How a client learns that the view changed

Three mechanisms, from simplest to most direct.

**Polling.** The client asks on a timer. It is simple, and it is wasteful: most polls return "nothing changed," and a change waits until the next poll.

**Query, then subscribe.** The client reads the current value, then watches for updates. Coordination services do this for cluster metadata: a member reads the current assignment and registers a watch so the next change is pushed. A product does the same thing with a websocket, with server-sent events, or with a push channel. The watch is the ancestor. The product version adds user credentials and **fan-out**: one view update becomes many pushes, one per connected client.

**Push when the view updates.** The job that maintains the materialized view notifies subscribers as part of applying the event. The client does not ask. The notification is the change.

## Correctness you can still claim

### The business invariant, not only the disk

Do not trust every hop to be perfect. A timeout retries the request. A double-click sends it twice. A replay sends it again. Put an **idempotency key** on the business operation so those duplicates cannot double-charge. The key lives on the write API: the client sends `Idempotency-Key: order-8841-attempt-1` with the charge, and the record stores that it already applied that key.

Atomicity inside one database is not enough, for two different failures. Two browser tabs can submit one human intent as two requests with **different** keys. The database dutifully applies both, because they do not look like duplicates. Or one key can be applied twice across a replay, if the handler does not record the key before it charges. End to end means the business invariant is enforced where the business knows enough — at the operation the user actually meant — not only inside one storage engine's transaction.

### A constraint checked after the fact

Uniqueness, and "the account has sufficient balance," can be checked **asynchronously**. A consumer reads the event, applies the rule, and may **reject after the fact**.

Two product patterns follow.

**Wait for the check.** The user sees a spinner until the consumer confirms. Latency is higher. Apologies are rarer, because you do not show success before the constraint has passed.

**Optimistic accept, then apologize.** You take the booking, and sometimes you send "we had to undo this." Airlines overbook seats on this pattern. It scales, because the request path does not wait for every downstream check. It needs a clear story for the user, and a **compensating transaction**: a later action that undoes the earlier one, such as a refund or a released hold.

This is often more operable than a distributed lock across services, or XA two-phase commit across them. It is not permission to corrupt money and sort it out later. A ledger still needs a sharp consistency story for the record of truth. The balance that is the books is not a cache you apologize for.

A wallet balance is the sharp case. It must not stay wrong. "People also bought" is the other pile. A stale recommendation is a timeliness problem. A balance that permanently disagrees with the ledger is an integrity problem.

### Stale for a while, wrong forever is a different bug

A derived view may be slightly **stale**. That is timeliness. It must not be **wrong forever**. That is integrity. **Linearizability** is the strong recency guarantee: the system behaves as if there were one copy, and once a write has completed, any read that starts later sees that write or something newer. Reserve it for the few facts that cannot be stale. A username that must be unique. A lock. The ledger line. Most of a social application is the other pile. The feed can lag.

Say both numbers. "Search may lag by 5 seconds" is the timeliness budget. "Search must not permanently miss a delete" is the integrity rule. A delete that arrives late and then sticks is acceptable. A delete that the index drops on the floor, so the document comes back in every future rebuild, is not.

### Assume drift, and go look for it

Derived systems drift. A bug in the function, a dropped message, a schema mismatch between producer and consumer. Budget for **detecting** that, not only for building the pipes.

An audit consumer reads the system of record and the derived store and compares them. Checksums catch bit-level disagreement. Row counts catch a missing partition. **Lineage** records which job produced this partition, so a bad build can be named. A canary record, one you know the right answer for, runs through the same pipes and is checked on the other side. Run one of these on a schedule. A weekly comparison of the search index against the orders table will catch a dual-write hole that no request-path metric sees.

## What a design states

Walk one order through the design and write down six facts.

1. **The system of record.** The order row commits in Postgres. That commit is what "the order exists" means.
2. **The derived stores, and how they are fed.** Search, the warehouse, the cache, features, and notifications consume change data capture or an explicit event. They are not written by the request handler.
3. **The idempotency key on the write API.** A retry of "place order 8841" hits the stored key and does not charge again. Two tabs that invent two keys are two intents, and the API will treat them as two.
4. **What may be stale, and for how long.** Search may lag by 5 seconds. The charge may not.
5. **How you rebuild after a bug.** Replay the log, or recompute from the lake, through the corrected function, into a new snapshot of the derived store.
6. **Which constraints are synchronous, which are asynchronous, and what the user sees on rejection.** Sufficient balance might be checked after the accept, with a compensating refund. Uniqueness of the payment id is checked on the record, in the request.

That list is the design. A page of product logos is not.

Replay is also what makes the system **evolvable**. You can add a new index, or change the feature function, and build it by replaying history. A dual write cannot do that. The second write was not kept. There is no input to run the new function on, except whatever happens to still be in each store, already transformed the old way.

## Recap

Specialized stores stay honest when one system of record accepts the write and every other store is a function of that write, fed by a durable ordered log or by files you can recompute. A second write inside the request is how those stores diverge. Stream and batch must implement the same function, or the nightly job fights the live one; one codepath, a shared definition, or a batch rebuild that heals drift are the ways out. Unbundling is a database's own modules — log, index, cache, view maintainer — with the network between them, which means you now own consistency, idempotency keys, and the choice of which constraints wait and which apologize. Timeliness is allowed lag. Integrity is the promise that a delete, or a charge, is not lost forever. Drift is expected, and an audit that compares the derived store to the record is part of the system.

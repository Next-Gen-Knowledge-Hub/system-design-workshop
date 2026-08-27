# 8. Transactions

Companion notes for **Chapter 8** of *Designing Data-Intensive Applications*
(2nd ed.). Transactions are a lie we ask the database to tell: “pretend
concurrent users and crashed processes did not smash the data.” When the
lie is too expensive, we pick a weaker lie (isolation level) and handle
the leftover bugs in the application.

## What this chapter is for

Not every app needs multi-row ACID. A key-value `GET`/`PUT` of one
object often does not. The moment you **read-modify-write**, update two
records that must stay in sync, or enforce “this seat is only sold
once,” you are in this chapter — even if you never type `BEGIN`.

Without transactions, every crash, timeout, and race becomes a different
inconsistency story (denormalized copies drift, counters lose increments,
two seats sell once). With them, a large class of faults collapses to
**abort and retry**.

## What exactly is a transaction?

A **transaction** groups several reads and writes into one logical unit
that either **commits** (all visible) or **aborts** (as if it never
happened). The application’s error handler becomes “retry the
transaction” instead of “manually unwind five tables.”

This idea is older than “NoSQL vs SQL.” Document stores, graph DBs, and
key-value engines that offer multi-key transactions are solving the same
problem the relational world named first.

### The meaning of ACID

The acronym is marketing glued to four different ideas. Be precise:

- **Atomicity** (here: *all or nothing* for writes in the transaction,
  not atomic CPU instructions). Abort rolls everything back. Partial
  failure inside the txn is the application’s problem only as “retry.”
- **Consistency** — the overloaded one. In ACID it means *your
  application invariants* (balance ≥ 0, every order has a payment row)
  hold **if each transaction preserves them**. The database cannot know
  your business rules unless you encode them as constraints. Do not
  confuse with *consistency* in CAP, replica consistency (ch. 6), or
  linearizability (ch. 10).
- **Isolation** — concurrent transactions do not step on each other.
  In theory, serializable. In practice, most databases default weaker
  and leave some races to you.
- **Durability** — after commit, a crash does not eat the write (WAL,
  fsync, replication — ch. 4 / 6). “Committed to the leader’s memory”
  is not durable. Sync replication changes what “durable” means across
  machines.

### Single-object and multi-object operations

**Single-object** operations (increment one counter with a compare-and-set,
atomic `PUT` of one document) are widely atomic even without a transaction
API. Many “NoSQL” stores give you that much.

**Multi-object** is where you need a real transaction — or a carefully
designed log + idempotency (ch. 12–13):

- Decrement inventory *and* create the order.
- Update a denormalized `user_name` on every post when the profile
  changes.
- Insert a row into `accounts` and into `account_balances`.

Foreign keys, secondary indexes, and materialized views are multi-object
under the hood. If the engine updates the row but crashes before the
index, you want atomicity across those structures — that is still a
transaction, even when the SQL never says `BEGIN`.

**Handling errors and aborts.** On abort (deadlock victim, serialization
failure, constraint violation, crash mid-txn), the app retries. Retries
must be **safe**: the business operation needs an idempotency story if
the client does not know whether commit succeeded (ch. 9 timeouts).

## Weak isolation levels

Vendors say “we have transactions” and then default to something that
still allows surprising anomalies. Learn the **anomalies**; the names of
levels follow. Weak levels are not “wrong” — they are cheaper, and they
push some races into application code.

| Anomaly | Picture | Read committed | Snapshot | Serializable |
|---|---|---|---|---|
| Dirty read | See someone else’s uncommitted write | prevented | prevented | prevented |
| Dirty write | Overwrite someone else’s uncommitted write | prevented (almost everywhere) | prevented | prevented |
| Read skew / nonrepeatable read | Two reads in one txn see different moments of the world | possible | prevented | prevented |
| Phantom | A search’s result set changes because of another txn’s insert | possible | usually prevented for plain reads | prevented |
| Lost update | Two read-modify-write; last commit wins; first increment dies | possible | depends on implementation | prevented |
| Write skew | Each txn’s write is fine *given what it read*; together they break an invariant | possible | **possible** | prevented |

### Read committed

You never see uncommitted data. Implementations often use **row locks**
for writers and brief locks or “read the latest committed version” for
readers. Still easy to lose updates (`read count, add 1, write count`
without `UPDATE … SET count = count + 1` or `SELECT FOR UPDATE`).
Still easy to see **read skew**: transfer money between accounts, a
concurrent reader sees one account after the debit and the other before
the credit — totals look wrong mid-flight.

This is a common default. Treat it as “no dirty reads,” not “safe for
inventory.”

### Snapshot isolation and repeatable read

Your reads come from a **consistent snapshot** — usually via **MVCC**:
old row versions hang around so readers do not block writers. Great for
long reads, reports, and backups (`pg_dump` style).

Names are messy: PostgreSQL’s `REPEATABLE READ` is close to snapshot
isolation; MySQL InnoDB’s `REPEATABLE READ` is not quite textbook SI
(gap locks change the story). Always check the product, not the SQL
standard label.

**Phantoms under SI:** a plain “read matching rows again” often sees the
same set because you are stuck to the snapshot. Phantoms that feed
**write skew** (decide based on a search, then write) need serializable
isolation or explicit range locks.

### Preventing lost updates

Two clients read `count = 10`, both write `11`. One increment vanished.

Fixes:

1. **Atomic update** in one statement: `SET count = count + 1`.
2. **Explicit lock**: `SELECT … FOR UPDATE`, then modify.
3. **Compare-and-set / optimistic version**: `UPDATE … WHERE version = ?`
   and retry if zero rows.
4. An isolation level / engine that **detects** lost updates under SI
   (some do, some don’t — verify).

If your API is get-modify-put on a document store, you own this race
unless the store offers conditional writes.

### Write skew and phantoms

**Write skew** is the anomaly SI does **not** fix. Classic picture: two
doctors on call; policy says at least one must remain. Each transaction
reads “two on call,” each sets itself off-call. Both commits succeed.
Each write touched a **different** row, so SI’s “first committer wins on
the same row” never fired. The invariant died.

Same shape: two remaining seats, two bookings, each saw `available = 1`.
Or: materialize “no overlapping meetings” by reading a range and
inserting — a **phantom** insert by the other txn breaks the premise.

Only **serializable** isolation (or application locks / constraints that
cover the predicate) prevents this class. Constraints help when you can
encode the invariant (`EXCLUDE` in Postgres, unique keys); they do not
replace thinking about the read-decide-write pattern.

## Serializability

**Serializable** means the result is the same as *some* serial order of
the transactions — as if they ran one after another. Three implementation
families dominate:

### Actual serial execution

One thread (or one partition) runs transactions one after another —
often as **stored procedures** so the whole txn is one round-trip
(VoltDB-style, Redis Lua, some H-Store descendants). Blazing if
transactions are short and data fits in memory / one shard. Dies if
you need long interactive multi-statement transactions across many
partitions: you either partition until each txn is single-shard, or you
leave this family.

### Two-phase locking (2PL)

Readers and writers take locks; writers block readers of that row (and
more, with range locks for phantoms). Hold until commit. Classic
serializability for decades. Under contention, latency explodes;
**deadlocks** abort a victim. Many apps avoid true 2PL serializable
because the throughput cliff is real.

### Serializable snapshot isolation (SSI)

Optimistic: run like SI, track read/write dependencies, **abort** at
commit if the execution was not serializable. Postgres `SERIALIZABLE`
is this family. Readers do not block writers the 2PL way. Aborts rise
under contention; **retries become part of the app**. SSI is often the
best default when you need serializability on a general-purpose RDBMS.

Pick based on contention, transaction length, and whether you can keep
work single-shard.

## Distributed transactions

Once a transaction touches **two shards or two systems**, atomicity is
no longer local. The failure mode is “left side committed, right side
did not.”

### Two-phase commit (2PC)

A **coordinator** runs:

1. **Prepare** — ask every participant to vote. A “yes” means: I have
   durably reserved the ability to commit or abort (locks + prepare
   record in the WAL). I will not forget this vote across crash.
2. **Commit or abort** — if all said yes, coordinator durably decides
   commit and tells everyone; if anyone said no or timed out, abort.

This is agreement on *whether this transaction happened*, not on a long
log of values (that is consensus / Raft — ch. 10). Related, different
shape.

**Costs:** locks held from prepare until outcome; if the coordinator dies
after prepare, participants are **in doubt** until recovery. Blocking.
Operator pain when the coordinator’s decision log is lost.

### Distributed transactions across different systems

**XA / heterogeneous 2PC** (Postgres + Kafka + Elasticsearch in one
txn) is operationally infamous: different prepare semantics, different
failure modes, application code driving the coordinator, and isolation
mechanisms that do not compose. The book’s tone: avoid as a cross-
product habit.

### Database-internal distributed transactions

Spanner, Cockroach, TiDB, Yugabyte, VoltDB-style engines: one vendor
owns participants, recovery, and often a consensus group per shard.
Still pay cross-shard latency; far less miserable than XA because
failover and “in doubt” are designed in.

### Exactly-once message processing revisited

“Consume a message, write to the DB, commit the offset” wants all-or-
nothing. Classic answer: 2PC between broker and DB. Practical answer
often:

- Store the **message id** (or offset) in the **same database
  transaction** as the side effects → processing becomes **idempotent**;
  retries cannot double-apply.
- Or **transactional outbox**: write business rows + outbox row
  atomically; a publisher drains the outbox.

Atomicity across *heterogeneous* broker + DB is not required if the DB
transaction alone makes the handler idempotent. Stream engines (Kafka
transactions, Kafka Streams EOS) specialize this further — ch. 12.

## How this shows up when you design something

- Money / inventory / seats: name the isolation level. “Read committed”
  is not enough for “don’t double-sell.”
- Read-modify-write: atomic update or version check, not get/put.
- Cross-shard transfer: database-internal 2PC, or a saga/outbox with
  idempotency — *say which*, and what the user sees on partial failure.
- Do not sprinkle `SERIALIZABLE` on every request “to be safe” without
  talking about abort rates and retry storms.
- Dual-write to search in the same request is not a transaction unless
  you have real atomic commit across both (usually you do not — ch. 13).

## Check yourself

1. Explain write skew with two on-call doctors (or two remaining seats)
   without using the word “lock.”
2. Why does snapshot isolation not automatically fix that?
3. Atomicity vs durability: a power cut after commit, vs abort in the
   middle of two updates — which ACID letter failed in each story?
4. When is 2PC the right tool, and when is an outbox + retry better?
5. Lost update vs write skew: same row or different rows? Which weak
   levels still allow each?
6. Why might serializable SSI abort a transaction that 2PL would have
   blocked instead?
7. You timed out waiting for `COMMIT`. Did the transaction commit?
   What must the client do?
8. Single-object CAS vs multi-object transaction: which one do you need
   to keep `orders` and `inventory` aligned?

Continue to [The trouble with distributed systems](../9-distributed-trouble/).

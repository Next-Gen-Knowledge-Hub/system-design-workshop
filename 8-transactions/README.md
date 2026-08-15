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

## What exactly is a transaction?

A **transaction** groups several reads and writes into one logical unit
that either **commits** (all visible) or **aborts** (as if it never
happened). The application’s error handler becomes “retry the
transaction” instead of “manually unwind five tables.”

### The meaning of ACID

The acronym is marketing glued to four different ideas. Be precise:

- **Atomicity** (here: *all or nothing* for writes in the transaction,
  not atomic CPU instructions). Abort rolls everything back.
- **Consistency** — the overloaded one. In ACID it means *your
  application invariants* (balance ≥ 0) hold if each transaction
  preserves them. The database cannot know your business rules unless
  you encode them as constraints. Do not confuse with *consistency* in
  CAP or replica consistency (ch. 6 / 10).
- **Isolation** — concurrent transactions do not step on each other.
  In theory, serializable. In practice, most databases default weaker.
- **Durability** — after commit, a crash does not eat the write (WAL,
  fsync, replication — ch. 4 / 6). “Committed to the leader’s memory”
  is not durable.

**Single-object** operations (increment one counter with a compare-and-set)
are widely atomic even without a transaction API. **Multi-object**
(decrement inventory *and* create order) is where you need a real
transaction — or a carefully designed log + idempotency (ch. 12–13).

## Weak isolation levels

Vendors say “we have transactions” and then default to something that
still allows surprising anomalies. Learn the anomalies; the names of
levels follow.

| Anomaly | Picture | Read committed | Snapshot | Serializable |
|---|---|---|---|---|
| Dirty read | See someone else’s uncommitted write | prevented | prevented | prevented |
| Dirty write | Overwrite someone else’s uncommitted write | prevented (almost everywhere) | prevented | prevented |
| Read skew / nonrepeatable read | Two reads in one txn see different snapshots of the world | possible | prevented | prevented |
| Phantom | A search’s result set changes because of another txn’s insert | possible | usually prevented for reads | prevented |
| Lost update | Two read-modify-write, last commit wins, first increment dies | possible | depends on implementation | prevented |
| Write skew | Each txn’s write is fine *given what it read*, together they break an invariant (two doctors both go off-call because each saw “one still on”) | possible | **possible** | prevented |

**Read committed:** you never see uncommitted data. Still easy to
lose updates (`read count, add 1, write count` without `UPDATE … SET
count = count + 1` or `SELECT FOR UPDATE`).

**Snapshot isolation / repeatable read:** your reads come from a
consistent snapshot (MVCC: old row versions hang around). Great for
long reads and backups. **Write skew** still happens because two
transactions can each update *different* rows based on the same
snapshot. Some engines protect lost updates under SI; some do not.
MySQL’s `REPEATABLE READ` is not quite textbook SI. Always check the
product.

**Preventing lost updates:** atomic increment, explicit locks, compare
and set (`UPDATE … WHERE version = ?`), or an isolation level that
detects them.

## Serializability

**Serializable** means the result is the same as *some* serial order of
the transactions. Three implementation families:

1. **Actual serial execution** — one thread, or one partition, runs
   transactions one after another (VoltDB-style, Lua in Redis,
   [redis scripting](../../redis-workshop/2-internals/README_Lua.md)).
   Blazing if transactions are short and data fits in memory. Dies if
   you need multi-partition interactive transactions.
2. **Two-phase locking (2PL)** — readers and writers take locks;
   writers block readers of that row. Classic serializability. Under
   contention, latency explodes. Deadlocks abort a victim.
3. **Serializable snapshot isolation (SSI)** — optimistic: run like SI,
   detect write-skew-ish conflicts at commit, abort the loser. Postgres
   `SERIALIZABLE` is this family. Aborts rise under contention; retries
   become part of the app.

Pick based on contention and whether you can keep transactions short.

## Distributed transactions

Once a transaction touches **two shards or two systems**, atomicity is
no longer local.

**Two-phase commit (2PC):** a coordinator asks all participants to
**prepare** (vote yes, durable), then **commit** or **abort**. If
everyone votes yes, commit. If anyone says no or times out, abort.
This is Joshi’s
[Two-Phase Commit](../../distributed-systems-workshop/3-partitioning/README.md)
— agreement on *whether this transaction happened*, not on a log of
values (that is consensus / Raft).

Costs: locks held during prepare; if the coordinator dies after
prepare, participants are **in doubt** until it returns (or an
operator / protocol recovers). Cross-**heterogeneous** systems
(DB + message queue + search) via XA are operationally infamous.
**Database-internal** distributed transactions (Spanner, Cockroach,
VoltDB) are less miserable because one vendor owns recovery.

**Exactly-once message processing** often *is* “consume, write DB,
commit offsets” as one transaction, or a transactional outbox. Kafka
transactions ([kafka-workshop](../../kafka-workshop/3-internals-and-reliability/RELIABILITY.md))
are a specialized form. Chapter 12–13 offer the *async, idempotent*
alternative when 2PC across products is not worth it.

## How this shows up when you design something

- Money / inventory / seats: name the isolation level. “Read committed”
  is not enough for “don’t double-sell.”
- Read-modify-write: atomic update or version check, not get/put.
- Cross-shard transfer: 2PC, or a saga/outbox with idempotency, and
  *say which*.
- Do not sprinkle `SERIALIZABLE` on every request “to be safe” without
  talking about abort rates.

## Ties to other workshops

- [2PC Go sample](../../distributed-systems-workshop/3-partitioning/)
- [mongo atomic](../../mongo-wokshop/3-aggregation_and_atomic/ATOMIC.md)
- [redis transactions / Lua](../../redis-workshop/2-internals/README_Operations.md)
- [Kafka exactly-once](../../kafka-workshop/3-internals-and-reliability/RELIABILITY.md)

## Check yourself

1. Explain write skew with two on-call doctors (or two remaining seats)
   without using the word “lock.”
2. Why does snapshot isolation not automatically fix that?
3. Atomicity vs durability: a power cut after commit, vs abort in the
   middle of two updates.
4. When is 2PC the right tool, and when is an outbox + retry better?

Continue to [The trouble with distributed systems](../9-distributed-trouble/).

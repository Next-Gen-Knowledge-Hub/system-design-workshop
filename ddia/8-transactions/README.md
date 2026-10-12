# 8. Transactions

These notes teach transactions from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What a transaction is

A **transaction** groups several reads and writes into one logical unit. It either **commits** — every write becomes visible together — or **aborts** — the database proceeds as if those writes never happened. The application’s error handler becomes “retry the transaction,” instead of “manually unwind five tables.”

Not every application needs a multi-row transaction. A key-value `GET` or `PUT` of one object often does not. The moment you read a value, modify it, and write it back, or you update two records that must stay in agreement, or you enforce “this seat is sold only once,” you are in transaction territory even if the code never says `BEGIN`.

Without that grouping, every crash, timeout, and race is a different inconsistency: denormalized copies drift, counters lose increments, two buyers get the same seat. With it, a large class of faults collapses to abort and retry.

The idea is older than the argument between “NoSQL” and “SQL.” Document stores, graph databases, and key-value engines that offer multi-key transactions are solving the same problem the relational world named first.

### The four letters of ACID

ACID is marketing glued onto four different ideas. Use the letters precisely.

**Atomicity**, in this setting, means all or nothing for the writes inside the transaction. It does not mean an atomic CPU instruction. If the transaction aborts, every write it did is rolled back. A partial failure inside the transaction is the application’s problem only as “retry.”

**Consistency**, inside ACID, means your application invariants still hold — balance is at least zero, every order has a payment row — *if each transaction preserves them*. The database cannot know the business rule unless you encode it as a constraint. This letter is not the “consistency” in the CAP theorem (what a replicated system promises during a network partition), not replica freshness, and not linearizability (a later read must see a completed write). Those are different promises that happen to share a word.

**Isolation** means concurrent transactions do not step on each other. In the strict form, the result matches some serial order: as if the transactions ran one after another. In practice most databases default to something weaker and leave some races to you.

**Durability** means a crash after commit does not eat the write. The engine records the commit in a write-ahead log, calls `fsync` so the log is on disk, and often replicates that log. “Committed in the leader’s memory” is not durable: a power cut loses it. Synchronous replication changes what “durable” means across machines, because the commit now waits until another machine has the log too.

Two stories, two letters. A power cut after a successful commit, and the write is gone: durability failed. An abort in the middle of two updates, and one update is visible while the other is not: atomicity failed.

### One object, or several

**Single-object** operations are widely atomic even without a transaction API. An increment of one counter with compare-and-set, or an atomic `PUT` of one document, is one of these. Many stores that do not offer multi-key transactions still give you this much.

**Multi-object** work is where you need a real transaction, or a carefully designed log plus idempotency (the same operation, applied twice, leaves the same result as applying it once):

- Decrement inventory and create the order.
- Update a denormalized `user_name` on every post when the profile changes.
- Insert a row into `accounts` and a row into `account_balances`.

Foreign keys, secondary indexes, and materialized views are multi-object under the hood. If the engine updates the row and crashes before the index, you want atomicity across those structures. That is still a transaction, even when the SQL never says `BEGIN`.

On abort — deadlock victim, serialization failure, constraint violation, crash mid-transaction — the application retries. Retries have to be safe. If the client timed out and does not know whether the commit succeeded, repeating the business operation blindly can apply it twice. The operation needs an idempotency key, or a transaction that records “this request id already ran.”

## Weak isolation

Vendors say “we have transactions” and then default to a level that still allows surprising anomalies. Learn the anomalies. The level names follow from them. A weak level is cheaper, and it pushes some races into application code.

| Anomaly | What the client observes | Read committed | Snapshot | Serializable |
|---|---|---|---|---|
| Dirty read | A read sees another transaction’s write that later aborts | prevented | prevented | prevented |
| Dirty write | A write overwrites another transaction’s write that is not committed yet | prevented almost everywhere | prevented | prevented |
| Read skew / nonrepeatable read | Two reads in one transaction see different moments of the world | possible | prevented | prevented |
| Phantom | A search’s result set changes because another transaction inserted a row | possible | usually prevented for a plain read | prevented |
| Lost update | Two read-modify-write sequences; the last commit wins and the first increment disappears | possible | depends on the implementation | prevented |
| Write skew | Each transaction’s write is valid given what it read; together they break an invariant | possible | possible | prevented |

### Read committed

You never see uncommitted data. Implementations often take row locks for writers, and either take a brief lock or “read the latest committed version” for readers.

Lost updates are still easy. Two clients both read `count = 10`, both add one, both write `11`. One increment vanished. `UPDATE … SET count = count + 1`, or `SELECT … FOR UPDATE` before the modify, closes that hole. A bare read, add, write does not.

Read skew is still easy. A transfer debits one account and credits another. A concurrent reader sees the debited account after the debit and the other account before the credit. The totals look wrong in the middle, even though both of the reader’s reads were of committed data.

Read committed is a common default. It means “no dirty reads.” It does not mean “safe for inventory.”

### Snapshot isolation and repeatable read

Under snapshot isolation, your reads come from a **consistent snapshot**: one moment of the database, fixed when the transaction started. The usual implementation is **MVCC** (multi-version concurrency control). Old row versions stay around so a reader can see the snapshot without blocking a writer who is creating a newer version. That is a good fit for long reads, reports, and backups in the style of `pg_dump`.

The names are messy. PostgreSQL’s `REPEATABLE READ` is close to snapshot isolation. MySQL InnoDB’s `REPEATABLE READ` is not textbook snapshot isolation; its gap locks change which phantoms can appear. Check the product. The SQL standard’s label is not the mechanism.

A plain “read the matching rows again” under snapshot isolation usually sees the same set, because you are stuck to the snapshot. A phantom that feeds write skew — you decide based on a search, then you write — is a different bug, and it needs serializable isolation or an explicit range lock.

### Stopping a lost update

Two clients read `count = 10`. Both write `11`. One increment is gone.

Four fixes:

1. An **atomic update** in one statement: `SET count = count + 1`. The engine reads and writes under its own lock, so the two increments stack.
2. An **explicit lock**: `SELECT … FOR UPDATE`, then modify, so the second transaction waits until the first commits.
3. **Compare-and-set**, also called an optimistic version check: `UPDATE … WHERE version = ?`. If zero rows change, someone else wrote first, and you retry.
4. An isolation level or engine that detects lost updates under snapshot isolation. Some engines do. Some do not. Verify the one you run.

If the API is get, modify, put on a document store, you own this race unless the store offers conditional writes.

### Write skew and phantoms

Write skew is the anomaly snapshot isolation does not fix.

Two doctors are on call. Policy says at least one must remain. Each transaction runs like this:

1. Read the on-call rows. The snapshot says two doctors are on call.
2. Conclude “I can leave, because the other doctor is still on call.”
3. Set *this* doctor off call.
4. Commit.

Both commits succeed. Each write touched a different row, so snapshot isolation’s “first committer wins on the same row” never fired. The invariant is dead: nobody is on call.

The same shape, with seats. Two seats remain, stored as `available = 1` on two different rows, or as a count each transaction reads and then decrements on its own booking row. Each booking reads `available = 1` and inserts its own booking. Both commit. The room is oversold.

The same shape, with calendars. Each transaction reads the range of meetings, sees no overlap, and inserts. The other transaction’s insert is a **phantom**: a row that was not in the snapshot the first transaction decided on. The premise of the insert is already false.

Serializable isolation prevents this class. So does an application lock or a constraint that covers the predicate, not just the row. A Postgres `EXCLUDE` constraint, or a unique key, helps when you can encode the invariant in the schema. Constraints do not replace thinking about the read-decide-write pattern. If the decision depends on a search, the thing you must protect is the search’s result, not only the row you happen to update.

## Serializable isolation

**Serializable** means the result is the same as some serial order of the transactions, as if they ran one after another. Three implementation families dominate.

### Actually running them one at a time

One thread, or one partition, runs transactions one after another. The transaction is often a **stored procedure**, so the whole unit is one round trip: VoltDB-style systems, Redis Lua scripts, and some descendants of H-Store. This is fast when transactions are short and the data fits in memory on one shard. It falls over when you need long, interactive, multi-statement transactions across many partitions. Either you partition until each transaction is single-shard, or you leave this family.

### Two-phase locking

**Two-phase locking (2PL)** has readers and writers take locks. A writer blocks readers of that row. Range locks extend the same idea to phantoms: a transaction that searched a range holds a lock that stops another transaction inserting into that range. Locks are held until commit. This was the classic implementation of serializability for decades. Under contention, latency explodes, and **deadlocks** — two transactions each waiting for a lock the other holds — abort one of them as the victim. Many applications avoid true 2PL serializable because that throughput cliff is real.

### Serializable snapshot isolation

**Serializable snapshot isolation (SSI)** is optimistic. Transactions run as they would under snapshot isolation. The engine tracks read and write dependencies and **aborts** at commit time if the execution was not equivalent to some serial order. PostgreSQL’s `SERIALIZABLE` is this family. Readers do not block writers the way 2PL readers do. Aborts rise under contention, and retries become part of the application.

SSI aborts a transaction that 2PL would have blocked instead. Under 2PL the second doctor’s read waits on the first doctor’s lock, so the second read sees the updated roster and the bad commit never starts. Under SSI both doctors read the snapshot, both try to commit, and the dependency check aborts one of them after the work was done. The application retries the aborted one.

Pick a family from the contention you expect, how long a transaction stays open, and whether you can keep the work on one shard. Putting `SERIALIZABLE` on every request “to be safe” raises the abort rate. Under load those retries become a retry storm: the failed transactions come back and contend again, and the abort rate climbs further.

## Transactions on more than one shard

Once a transaction touches two shards, or two systems, atomicity is no longer local. The failure is: the left side committed, the right side did not.

### Two-phase commit

A **coordinator** runs **two-phase commit (2PC)**:

1. **Prepare.** The coordinator asks every participant to vote. A “yes” means: I have durably reserved the ability to commit or abort. Locks are held, and a prepare record is in the write-ahead log. A crash will not make me forget this vote.
2. **Commit or abort.** If every participant said yes, the coordinator durably decides commit and tells everyone. If anyone said no, or timed out, the coordinator decides abort.

This is agreement on whether *this transaction* happened. It is a different shape from agreeing a long log of values, which is what a consensus protocol such as Raft does: a replicated sequence of commands that survives the loss of a leader. The two ideas are related. They are not the same protocol.

The costs are concrete. Locks stay held from prepare until the outcome is known. If the coordinator dies after prepare, participants are **in doubt** until recovery finds the coordinator’s decision. The protocol blocks. If the coordinator’s decision log is lost, an operator is stuck with participants that will neither commit nor abort on their own.

### Across different products

**XA**, heterogeneous two-phase commit — Postgres and Kafka and Elasticsearch in one transaction — is operationally infamous. Prepare means something different in each product. Failure modes differ. Application code often drives the coordinator. Isolation mechanisms do not compose: a lock in one system does not see a snapshot in another. Avoid this as a habit across products.

### Inside one database

Spanner, CockroachDB, TiDB, Yugabyte, and VoltDB-style engines run distributed transactions where one vendor owns the participants, the recovery, and often a consensus group per shard. You still pay cross-shard latency. Failover and the in-doubt state are designed in, which is a different operational life from XA.

### A message that must apply once

“Consume a message, write to the database, commit the offset” wants all-or-nothing. If the database write commits and the offset commit does not, the message is processed again. If the offset commits and the database write does not, the message is lost.

The classic answer is two-phase commit between the broker and the database. The practical answers are usually different:

- Store the **message id** (or the offset) in the **same database transaction** as the side effects. Processing becomes idempotent. A retry cannot double-apply, because the second attempt sees the id and stops.
- Or use a **transactional outbox**. The business rows and an outbox row commit in one database transaction. A publisher drains the outbox and sends the messages. The send can be retried; the database transaction already made the intent durable.

Atomicity across a heterogeneous broker and database is unnecessary if the database transaction alone makes the handler idempotent. Stream engines — Kafka transactions, Kafka Streams exactly-once semantics — specialize this further inside one log system.

A saga is the other explicit choice for a cross-shard transfer: a sequence of local transactions with a compensating action when a later step fails. Say which one you are using, database-internal two-phase commit or saga plus outbox plus idempotency, and say what the user sees when the first shard has committed and the second has not.

Writing the row to the database and the document to a search index in the same request is a dual-write. It is not a transaction unless both sides share a real atomic commit. Usually they do not: the database commits, the search call fails or lands later, and the index drifts from the row.

## Mechanism recap

- A transaction commits every write together or aborts them all. Atomicity is that all-or-nothing rollback. Durability is a committed write surviving a crash via the write-ahead log, `fsync`, and often replication. ACID consistency is your invariants, held only if each transaction preserves them. Isolation is how much concurrent transactions are allowed to see of each other.
- Read committed blocks dirty reads and dirty writes, and still allows read skew, lost updates, phantoms, and write skew. Snapshot isolation (MVCC) gives each transaction one snapshot, which stops read skew and usually stops a plain phantom read, and still allows write skew when two transactions write different rows.
- A lost update on one counter is closed by `SET count = count + 1`, `SELECT … FOR UPDATE`, or `UPDATE … WHERE version = ?`. Write skew — two on-call doctors, two remaining seats, two overlapping meetings — needs serializable isolation, or a constraint that covers the predicate (`EXCLUDE`, a unique key).
- Serial execution (stored procedures, VoltDB, Redis Lua, H-Store) runs one transaction at a time on a shard. Two-phase locking holds read and write locks until commit and deadlocks under contention. Serializable snapshot isolation (PostgreSQL `SERIALIZABLE`) runs like a snapshot and aborts at commit when the result was not serializable.
- Two-phase commit has a coordinator collect durable prepare votes, then commit or abort. Participants block if the coordinator dies in doubt. XA across products is a poor habit. Inside one database, or via an idempotent message id and a transactional outbox, the same “both sides or neither” goal has a recovery story you can operate. A dual-write to search in the same request is not that commit. `SERIALIZABLE` on every request trades abort rate for a retry storm.

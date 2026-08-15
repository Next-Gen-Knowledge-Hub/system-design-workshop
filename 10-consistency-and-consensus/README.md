# 10. Consistency and consensus

Companion notes for **Chapter 10** of *Designing Data-Intensive Applications*
(2nd ed.). This chapter was largely rewritten for the second edition.
The question is: when there are several copies of data (ch. 6) on
unreliable nodes (ch. 9), what does **“the” value** even mean — and
which algorithms can make a cluster behave like one careful computer?

Hands-on:
quorum, Paxos, Raft.

## What this chapter is for

“Strong consistency” is a cloud-brochure phrase. The precise word you
usually want is **linearizability**. The precise tool that implements
it on a cluster that can lose nodes is **consensus** (Raft, Paxos,
Zab, …). Many databases that *look* replicated are not linearizable.
Knowing the difference is the whole chapter.

## Linearizability

A system is **linearizable** if it behaves as if there were **one copy**
of the data, and every operation takes effect **atomically at a single
point in time** between its start and finish. If client A completes a
write of `x=1`, then client B starts a read, B must see `1` (or a later
value). No “it depends which replica.”

That is stricter than:

- **Serializability** (ch. 8) — about *transactions* behaving as some
 serial order; it does not by itself say replicas exist.
- **Causal / sequential / eventual consistency** — weaker stories that
 many geo systems actually provide.

**What you use linearizability for:** uniqueness (“this username is
taken”), locks and leader election, single-object compare-and-set,
“read the thing I just wrote” across clients, some ID guarantees.

**Cost:** if the network between replicas is slow, *every* linearizable
op waits. In a geo deployment, that can be tens of milliseconds to
seconds. CAP in practice: during a partition, a linearizable system
must refuse some operations (usually writes, sometimes reads) rather
than return a guess. Systems that stay available in both partitions
are **not** linearizable.

**Implementing it:** a single-thread single-node store is linearizable
(until it dies). A replica set is linearizable only if reads and writes
go through a protocol that cannot serve stale data (leader + quorum
commit + **linearizable reads** that do not lie from a stale leader —
often “read from leader with a lease” or “quorum read”). Async
primary/replica is **not** linearizable. Dynamo-style quorum is **not**
automatically linearizable.

## ID generators and logical clocks

`AUTOINCREMENT` on one node is linearizable and a single point of
failure. Distributed IDs:

- UUIDs: unique, not ordered, not causal.
- Snowflake-style (timestamp + worker id): mostly ordered, **not**
 linearizable, clock-dependent (ch. 9).
- Pre-allocated blocks: fast, can have gaps.

**Logical clocks** (Lamport clocks, hybrid logical clocks)
give an order that respects **happens-before**. They do **not** give
linearizable IDs by themselves. If you need “this id was assigned
before that one *in real time across the cluster*,” you are back to
consensus or a single allocator.

## Consensus

Informal: a group of nodes **agrees on a value** (or on a **sequence**
of values) such that they will not change their minds, even if some
nodes crash and messages duplicate.

**Total order broadcast / replicated log:** agree on the next log entry,
forever. Raft and Multi-Paxos are this. Kafka’s controller, etcd,
ZooKeeper, Consul, and the commit log of many “NewSQL” stores are this.
Once you have a replicated log, you can run a state machine on it
(a **replicated log** driving a state machine).

Problems that **are** consensus in disguise (if you can solve one, you
can solve the others, with care):

- Linearizable compare-and-set
- Locks and leases (who owns it)
- Uniqueness constraints under concurrency
- Append to a shared log (total order)
- Atomic commit (all shards yes or all no) — related, with extra
 operational shape via 2PC (ch. 8)

**Consensus in practice:** you run it on a **small** cluster (3 or 5
nodes), not on 500. The 500 get the decision via replication or gossip.
That small cluster is the **consistent core** for membership, leader,
config, and partition maps — not your petabytes of user rows.
etcd/ZooKeeper are the usual implementations.

**Coordination services** (ZK, etcd): ephemeral nodes, watches /
state notifications, and leader election. Use them as a core, not as
a general database.

What consensus does **not** fix: Byzantine lying (usually), your
application’s write skew across two keys (need transactions), or
“I timed out so I will retry with a new id” without idempotency.

## How this shows up when you design something

- Need “only one owner of this job”? Consensus or a lease with fencing
 from a consensus store — not a single Redis `SETNX` without a token.
- Need “read your writes” for one user? Often sticky sessions or read
 from primary; you do not need linearizability of the whole social
 graph.
- Need unique username at 10k QPS globally? Linearizable index or an
 allocation service; eventual uniqueness with “sorry, pick another”
 is a product choice (ch. 13).
- Draw etcd as a tiny box. Do not put user traffic through it.

Interview phrase: “This constraint is linearizable; this feed is
eventual; here is why that split is OK.”

## Check yourself

1. After a linearizable write of `x=1` completes, a read that started
 later returns `0`. What invariant broke?
2. Why can a majority-quorum key-value store still show a value that
 then disappears?
3. Why is “unique email” a consensus-shaped problem and “count of likes”
 usually not?
4. Why 5 etcd nodes, not 50?

Continue to [Batch processing](../11-batch-processing/).

# 10. Consistency and consensus

Companion notes for **Chapter 10** of *Designing Data-Intensive Applications*
(2nd ed.). This chapter was largely rewritten for the second edition.
The question is: when there are several copies of data (ch. 6) on
unreliable nodes (ch. 9), what does **“the” value** even mean — and
which algorithms can make a cluster behave like one careful computer?

## What this chapter is for

“Strong consistency” is a cloud-brochure phrase. The precise word you
usually want is **linearizability**. The precise tool that implements
it on a cluster that can lose nodes is **consensus** (Raft, Paxos, Zab,
…): single-leader replication **done right**, with automatic failover
that does not lose committed data or split-brain.

Many databases that *look* replicated are not linearizable. Knowing the
difference is the whole chapter.

## Linearizability

### What makes a system linearizable?

A system is **linearizable** if it behaves as if there were **one copy**
of the data, and every operation takes effect **atomically at a single
point in time** between its invocation and its response. Once a write
of `x=1` **completes**, any read that starts later must see `1` (or a
later value). No “it depends which replica.”

That is a **recency** guarantee for a single object (or a linearizable
register / CAS). It is stricter than:

- **Serializability** (ch. 8) — about *transactions* matching some
  serial order; it does not by itself talk about replicas or real-time
  ordering across clients.
- **Causal / sequential / eventual consistency** — weaker stories that
  many geo systems actually provide.
- **“W+R>N quorum”** (ch. 6) — overlap helps freshness but does not by
  itself give linearizability under concurrency and timing races.

You can draw linearizability with a timeline: each op sits on one point
inside its interval; those points must respect real-time order for
non-overlapping ops.

### Relying on linearizability

You reach for it when the product needs a **single truth** now:

- Uniqueness (“this username is taken”).
- Locks, leases, and leader election.
- Single-object compare-and-set.
- Cross-client “read the thing that just finished writing.”
- Some ID allocation stories (below).

You do **not** need it for a social feed, “people also bought,” or most
analytics. Claiming linearizability for the whole graph is how designs
become slow and fragile.

### Implementing linearizable systems

A single-thread single-node store is linearizable until it dies. A
replica set is linearizable only if the protocol cannot serve stale
data as “the” answer:

- Writes committed by a **quorum** (or sync to a majority).
- Reads that cannot return from a stale ex-leader: **leader lease**,
  **quorum read**, or read-your-writes tricks that are carefully proven.
- Async primary/replica is **not** linearizable.
- Dynamo-style quorum is **not** automatically linearizable (ch. 6).

Consensus (below) is the usual way to get a linearizable replicated
log; then the state machine on top inherits the order.

### The cost of linearizability

If the network between replicas is slow, *every* linearizable op waits
for the protocol. In a geo deployment, that can be tens of milliseconds
to seconds. CAP in practice: during a partition, a linearizable system
must **refuse** some operations (usually writes, sometimes reads) rather
than return a guess. Systems that accept writes in both partitions are
**not** linearizable for those writes.

ZooKeeper / etcd stay available for quorum members; the minority
partition blocks. That is the trade, named honestly.

## ID generators and logical clocks

`AUTOINCREMENT` on one node is linearizable and a single point of
failure. Distributed IDs trade properties:

| Approach | Unique? | Ordered? | Linearizable? | Notes |
|---|---|---|---|---|
| UUID v4 | yes | no | no | Safe, opaque, bad as clustered PK |
| Snowflake-style | yes | mostly by time | no | Clock-dependent (ch. 9); worker id |
| Pre-allocated blocks | yes | per allocator | no | Gaps on crash; fast |
| Consensus / single allocator | yes | yes | can be | Bottleneck; correct |

### Logical clocks

**Lamport clocks**, **version vectors**, **hybrid logical clocks** give
an order that respects **happens-before** (causality). They do **not**
give linearizable real-time IDs by themselves. They shine for detecting
concurrency (ch. 6) and for causal consistency.

If you need “this id was assigned before that one *in real time across
the cluster*,” you are back to consensus or a single allocator — a
**linearizable ID generator**.

## Consensus

Informal: a group of nodes **agrees on a value** (or on a **sequence**
of values) such that they will not change their minds, even if some
nodes crash and messages duplicate. Safety over partitions; liveness
when a majority can talk.

### The many faces of consensus

Problems that **are** consensus in disguise (reduce one to another with
care):

- Agreeing on the next **log entry** (**total order broadcast** /
  replicated log).
- Linearizable **compare-and-set**.
- **Leader election** and locks / leases (who owns it).
- Uniqueness under concurrency.
- **Atomic commit** (all shards yes or all no) — related to 2PC (ch. 8);
  2PC alone is not full consensus if the coordinator is a single point,
  which is why serious systems combine ideas carefully.

**Raft** and **Multi-Paxos** implement a replicated log: clients propose
commands; the leader commits an entry once a majority persists it;
followers apply in order. Kafka’s controller epoch story, etcd,
ZooKeeper (Zab), Consul, and the commit path of many NewSQL stores sit
here. Once you have the log, you run a **state machine** on it
(replicated state machine): the log is the truth; the state is derived.

### Consensus in practice

You run consensus on a **small** cluster (typically 3 or 5 voters), not
on 500. The 500 get decisions via replication, gossip, or by reading
the small core’s output. That small cluster is the **consistent core**
for membership, leader, config, and shard maps — not your petabytes of
user rows.

Properties you actually buy with Raft-ish failover:

- No split-brain leaders for the same term / epoch.
- No loss of **majority-committed** entries after failover.
- Automatic leader election when the old leader is partitioned away.

Properties you do **not** buy: Byzantine tolerance (usually),
serializable multi-key transactions (need ch. 8 machinery on top), or
“timeout means the command did not apply” (still need idempotency).

### Coordination services

ZooKeeper, etcd, Consul: small, strongly consistent stores optimized
for **coordination**, not bulk data.

Typical primitives:

- Linearizable reads/writes of small keys.
- **Ephemeral** nodes / leases for membership.
- **Watches** / notify-on-change (observe derived cluster state).
- Leader election recipes built on those.

Use them as a core for “who owns this shard,” not as the user database.
Putting high-QPS session traffic through etcd is how you discover its
throughput ceiling.

## How this shows up when you design something

- Need “only one owner of this job”? Consensus or a lease **with
  fencing** from a consensus store — not a single Redis `SETNX`
  without a token.
- Need “read your writes” for one user? Sticky sessions or read from
  primary often enough; you do not need linearizability of the whole
  social graph.
- Need unique username at global QPS? Linearizable index / allocation
  service, or eventual uniqueness with “sorry, pick another” as a
  product choice (ch. 13).
- Draw etcd as a tiny box. Do not put user traffic through it.
- Interview phrase: “This constraint is linearizable; this feed is
  eventual; here is why that split is OK.”

## Check yourself

1. After a linearizable write of `x=1` completes, a read that started
   later returns `0`. What invariant broke?
2. Why can a majority-quorum key-value store still show a value that
   then disappears?
3. Why is “unique email” a consensus-shaped problem and “count of likes”
   usually not?
4. Why 5 etcd nodes, not 50?
5. Serializability vs linearizability: which one is about transactions,
   which about real-time single-object recency?
6. Snowflake IDs are unique and roughly ordered. Why are they still not
   a linearizable ID generator?
7. What does a fencing token have to do with leader election after a
   pause (ch. 9)?
8. During a network partition, a linearizable register must do what
   with writes in the minority side — and why?

Continue to [Batch processing](../11-batch-processing/).

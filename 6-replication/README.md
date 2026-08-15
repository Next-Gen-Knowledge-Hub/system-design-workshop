# 6. Replication

Companion notes for **Chapter 6** of *Designing Data-Intensive Applications*
(2nd ed.). One copy of the data is not enough: disks die, zones die,
users live on other continents, reads outgrow one box. Replication
means **the same data on several nodes**. The idea is simple. The
failure modes are not.

This chapter pairs naturally with
[distributed-systems-workshop replication](../../distributed-systems-workshop/2-replication/)
(WAL, quorum, Raft) and with
[mongo replication](../../mongo-wokshop/4-replication_and_sharding/REPLICATION.md).

## What this chapter is for

Name *why* you replicate, then pick a style that actually buys that
why:

| Goal | What replication can do |
|---|---|
| High availability | Keep serving when a node / zone / region dies |
| Durability | Survive a machine that never comes back |
| Latency | Put a copy near the user |
| Read scale | Serve reads from replicas |
| Disconnected / local-first | Keep working when the network does not |

Then pick **who is allowed to accept writes**: one leader, several
leaders, or no leader.

## Single-leader replication

One node is the **leader** (primary). All writes go there. **Followers**
(replicas, secondaries) replay a stream of changes. Reads may go to
the leader (always up to date, bottleneck) or to followers (scale,
risk of **lag**).

This is Postgres streaming replica, MySQL, Mongo replica set, Kafka
partition leader, most “boring” production defaults. It is popular
because humans can reason about it: there is one order of writes.

**Synchronous vs asynchronous.** Sync: leader waits until a follower
(or a quorum) has the write. Stronger durability; write latency follows
the slow replica; if that replica dies, writes stall unless you
reconfigure. Async: leader acknowledges after local persist. Fast;
failover can **lose** the last acknowledged writes. Many systems offer
“wait for N replicas” in the middle (Kafka `acks=all` +
`min.insync.replicas` — see
[kafka reliability](../../kafka-workshop/3-internals-and-reliability/RELIABILITY.md)).

**New followers:** copy a snapshot, then catch up the replication log
from the snapshot’s position. If you cannot do that, you cannot grow
the cluster.

**Node outages:** follower dies → catch up from the log (need enough
retention). Leader dies → failover. Failover needs: detect death
(timeouts, ch. 9), elect a new leader (ch. 10), make sure the old
leader cannot keep writing (**fencing**, generation clock — Joshi’s
[Generation Clock](../../distributed-systems-workshop/2-replication/README_Consensus.md)).
Split-brain is the nightmare: two leaders, diverging data.

**Replication log implementations:** statement-based (replay SQL — breaks
on nondeterminism), WAL shipping (physical), logical (row change
events). Logical change events are also how CDC feeds Kafka (ch. 12).

### Problems with replication lag

If you read from a follower:

- **Read-your-writes** can fail: you post, refresh, the replica has
  not got it yet.
- **Monotonic reads** can fail: you see a new value, then an older
  replica, time appears to go backward.
- **Consistent prefix** can fail: you see the reply before the question
  if those writes were on different partitions / arrived out of order.

Fixes are application-level as much as database-level: read from the
leader for “my own writes,” session stickiness, “read at least this
timestamp / LSN,” or don’t use lagging replicas for those paths.
Follower reads are a pattern with a contract — see
[Follower Reads](../../distributed-systems-workshop/2-replication/README_ClientHandling.md).

## Multi-leader replication

Several nodes accept writes and replicate to each other. Typical reason:
**two datacenters**, each wants local writes; a single global leader
would make one continent slow. Also: collaborative apps, and **sync
engines**.

Topologies: circular, star, all-to-all. Sparse topologies have single
points of failure in the *replication path*. All-to-all can deliver
writes **out of causal order** (update arrives before insert). Version
vectors exist to detect that; many products do not use them well.
Autoincrement keys, uniqueness constraints, and triggers become
surprising. The book’s tone is: **avoid multi-leader unless you must.**

**Conflict resolution:** last-write-wins (silent data loss), merge in
the app, CRDTs, or “store siblings and ask the user.” There is no
magic. If two leaders accepted different values for the same key, the
system must pick or combine.

### Sync engines and local-first (2nd edition)

Treat the **client device** as another replica. The user edits offline;
a sync engine (CRDTs, Replicache-like, Electric, Automerge, …) merges
later. This is multi-leader with millions of flaky leaders. It is the
right model for some products (notes, docs, field apps). It is the
wrong model for a bank ledger. Local-first also changes the privacy
story (data lives on the device — ch. 1 / 14).

## Leaderless replication

Dynamo, Cassandra, Riak, DynamoDB-ish designs: the client (or a
coordinator) writes to several nodes, reads from several nodes, uses
**quorums**. If `W + R > N`, the read set and write set overlap, so
you often see the latest value — **until** clocks, sloppy quorums, and
concurrent writes enter the chat.

You can write even if a node is down (`W=2, N=3`). A **hinted handoff**
or anti-entropy (read repair, Merkle trees) catches the missed node
up later.

**Performance vs single-leader:** you skip leader failover pauses, but
you pay coordination on *every* write and you get weaker consistency
unless you are very careful. The Joshi workshop’s
[Majority Quorum](../../distributed-systems-workshop/2-replication/README_Consensus.md)
note is the one to reread: quorum RW **alone is not linearizable**.
A value can appear and then vanish on the next read.

**Detecting concurrent writes:** last-write-wins by timestamp (lies when
clocks lie), version vectors (detect siblings — same as
[Version Vector](../../distributed-systems-workshop/2-replication/README_Versioning.md)).
Merging siblings is application work.

## How this shows up when you design something

Default sketch for a user-facing OLTP store:

1. Single-leader in one region, async replica for failover / reads.
2. Say what happens on failover (RPO / RTO).
3. If you need multi-region writes, *name the conflicts*.
4. If you pick Cassandra-style, *name the quorum* and what stale means.

Do not draw three boxes labeled “replica” without saying who takes
writes.

## Ties to other workshops

- [distributed-systems-workshop](../../distributed-systems-workshop/2-replication/)
  — WAL, quorum, Raft, version vectors in `Go`.
- [mongo-wokshop replication](../../mongo-wokshop/4-replication_and_sharding/REPLICATION.md)
- [kafka-workshop](../../kafka-workshop/3-internals-and-reliability/) —
  leader per partition, ISR, high watermark.

## Check yourself

1. You acknowledged a write with async replication and then the leader
   died. What might a client have seen that is now gone?
2. Give a read-your-writes bug in a social app, and one fix that does
   not require linearizability everywhere.
3. Why is last-write-wins dangerous for a shopping cart?
4. When would you actually want multi-leader, and what will you do with
   conflicts?

Continue to [Sharding](../7-sharding/).

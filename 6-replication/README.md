# 6. Replication

Companion notes for **Chapter 6** of *Designing Data-Intensive Applications*
(2nd ed.). One copy of the data is not enough: disks die, zones die,
users live on other continents, reads outgrow one box. Replication
means **the same data on several nodes**. The idea is simple. The
failure modes are not.

This chapter is where DDIA names the replication styles you will meet
in every production database: single-leader, multi-leader, and
leaderless (quorum) — plus the lag, failover, and conflict problems
that follow. The 2nd edition also treats **sync engines / local-first**
as multi-leader taken to the extreme: the user’s device is a replica.

## What this chapter is for

Name *why* you replicate, then pick a style that actually buys that
why. Replication is not “high availability” by itself. It is a
mechanism. What you get depends on **who may accept writes**, and
whether the leader **waits** for a replica before it acknowledges.

| Goal | What replication can do | What it cannot do |
|---|---|---|
| High availability | Keep serving when a node / zone / region dies | Save you from a bad deploy of the same binary |
| Durability | Survive a machine that never comes back | Undo `DROP TABLE` (that replicates too) |
| Latency | Put a copy near the user | Make a transcontinental *write* free if the leader is far |
| Read scale | Serve reads from replicas | Scale **writes** — they still hit every copy |
| Disconnected / local-first | Keep working when the network does not | Avoid merge conflicts on concurrent edits |

Then pick **who is allowed to accept writes**: one leader, several
leaders, or no leader.

Older docs say *master / slave*. Same idea as leader / follower; use
the newer words.

## The mental model

```
  Client writes
        |
        v
   ┌──────────┐     change stream      ┌──────────┐
   │  LEADER  │ ─────────────────────▶ │ FOLLOWER │
   │ (writes) │                        │  (replay)│
   └──────────┘                        └──────────┘
        |                                    |
        +── reads: always fresh              +── reads: scale, maybe lag
```

Every style is a variation on “copy a **log of changes** to other
nodes, and replay it.” The styles differ in:

1. How many nodes may **append** to that log.
2. Whether the writer waits for the copy (**sync**) or not (**async**).
3. What happens when two writers disagree.

Sharding (ch. 7) is the other axis: **different** data on different
nodes. In production you almost always combine them — each shard is
its own replicated group.

## Single-leader replication

One node is the **leader** (primary). All writes go there. **Followers**
(replicas, secondaries) replay a stream of changes. Reads may go to
the leader (always up to date, bottleneck) or to followers (scale,
risk of **lag**).

This is Postgres streaming replica, MySQL, Mongo replica set, Kafka
partition leader, most “boring” production defaults. It is popular
because humans can reason about it: there is **one order of writes**.

Failover is usually automatic in managed products, but it still rests
on detecting death (timeouts — ch. 9) and agreeing who is the new
leader (consensus — ch. 10).

### Synchronous versus asynchronous

**Synchronous:** the leader waits until a follower (or a configured
set) has the write on durable storage before it tells the client
“committed,” and usually before it shows that write to other readers.
You gain: if the leader dies, a follower has the last ack’d write.
You pay: write latency includes the replica’s disk and the network
to it. If that replica is slow or dead, **writes stall** unless you
drop it from the sync set.

**Asynchronous:** the leader acknowledges after *its own* persist. Fast
and available. Failover can **lose** the last acknowledged writes —
the client saw success; the new leader never saw the record. There
is no bound on lag: milliseconds on a quiet LAN, minutes or hours
under load, after a replica restart, or across a bad WAN link.

**Semi-sync / “wait for N”** sits in the middle: wait for *at least
one* (or `min.insync.replicas`) follower, not all of them. Kafka’s
`acks=all` plus a min-ISR is this idea on a log. You still choose
an **RPO** (how much acknowledged data you may lose) and an **RTO**
(how long promotion takes).

A useful thought experiment from the book: if the network between
two regions dies, a *synchronous* replica in the far region makes
the local leader unable to commit. That is single-leader with a
remote veto, not “two active regions.”

### Setting up new followers

You cannot just `rsync` a running data directory and hope. The copy
must be a **consistent snapshot** plus a position in the replication
log:

1. Snapshot the leader (or a caught-up follower) at a known log
   sequence / LSN / binlog coordinate.
2. Copy the snapshot to the new node.
3. The new follower connects and asks for **every change after that
   position**, then joins the live stream.

If the leader has already discarded the log past that position, the
new node cannot catch up and you start over from a newer snapshot.
This is why **log retention** is an operational knob, not a detail.

If you cannot snapshot + catch up, you cannot grow the cluster, you
cannot replace a disk, and you cannot clone a replica for analytics.

### Handling node outages

**Follower dies or is partitioned.** It keeps a local copy of the
changes it has already applied. On return it requests the gap from
the leader and catches up. Conceptually easy; operationally expensive
if the replica was down a long time or the write rate is high: both
leader and recovering follower burn I/O on the backlog.

The leader wants to **delete** old log once every follower has it.
An absent follower forces a choice: keep the log (disk risk on the
leader) or drop it (that follower must be rebuilt from backup).
Replication slots in Postgres are this contract made explicit.

**Leader dies.** Failover:

1. **Detect** death — timeouts. Too long: users wait. Too short: a
   GC pause or a network blip triggers a needless failover, which
   makes a sick cluster sicker.
2. **Elect** a new leader. With async replication, pick the follower
   with the **highest log sequence**. Promoting a replica that is
   days behind is a silent disaster.
3. **Reconfigure** clients and remaining followers to the new leader.
4. **Fence** the old leader so it cannot keep accepting writes if it
   comes back (split-brain). Typical tool: a monotonically increasing
   **generation / term / epoch**. Writes from an old generation are
   rejected. Without fencing, two leaders diverge and you merge by
   hand — or you do not notice until checksums fail.

Some teams still **fail over by hand** even when the software can do
it automatically, because a false failover under load is worse than
a few extra minutes of unavailability. Automatic failover is not
free HA; it is a policy with failure modes.

### Implementation of replication logs

How the stream is encoded matters as much as who leads.

| Style | What is shipped | Breaks when |
|---|---|---|
| **Statement-based** | Replay the SQL (`UPDATE …`) | `now()`, `rand()`, triggers, row-by-row differences, non-deterministic UDFs |
| **WAL / physical** | Redo bytes the storage engine already wrote | Engine or major-version mismatch; hard to decode as events |
| **Logical / row-based** | “this row became this” change events | Slightly more CPU; you must still order and filter them |

Physical WAL shipping is what Postgres streaming replication *is*.
Logical decoding is what lets you replicate a subset of tables, or
feed **CDC** into Kafka (ch. 12). Statement-based looks simple and
is a museum of edge cases; treat it as a historical warning, not a
design to copy.

Trigger-based “replication” (write a change table from triggers)
exists in old products. It is an application-level log with extra
failure modes. Prefer the engine’s logical stream if you have one.

### Problems with replication lag

Follower reads are a **performance** feature with a **consistency**
cost. Three session bugs show up constantly in product code:

**Read-your-writes (read-after-write) can fail.** You post, the app
redirects to the post page, the replica you hit has not got it yet.
The user thinks the submit failed and clicks again.

**Monotonic reads can fail.** You see the new bio, refresh, hit an
older replica, the bio is the old one. Time went backward. Users
report “the site is haunted,” not “eventual consistency.”

**Consistent prefix can fail.** A reply is visible before the message
it replies to, because those rows live on different shards with
different replica lag. Single-leader on **one** shard applies writes
in one order, so this does not happen *inside* that shard. Across
independent shards there is no global order unless you invent one
(causal metadata — later in this chapter and in ch. 10).

None of these require a full network partition. They happen on a
healthy cluster with 200 ms of lag.

### Solutions for replication lag

Do not pretend async replication is sync. Design the product for the
lag you will actually see when a replica is rebuilding, not the lag
on the dashboard at noon.

Application-level mitigations (you will write these even with a
“boring” RDS):

- Read **this user’s writes** from the leader (or a sync replica).
  Session stickiness: after a write, pin that session to the leader
  for a few seconds.
- **Read at least this timestamp / LSN** — wait until the replica’s
  applied position passes the write’s position (Bitbucket and others
  have published this pattern).
- Put causally related rows on the **same shard** so they share one
  log order.

Those are easy to get wrong. The programming model that makes them
go away is a database that actually gives **linearizability** (ch. 10)
and transactions (ch. 8) across the replicas you read — the NewSQL
generation (Cockroach, Spanner-like, TiDB, …). You pay coordination
and you still have to understand regions.

Weaker replication is still a valid pick when you want to **keep
writing through a network blip** or you cannot afford a consensus
round on every commit. The rest of the chapter is those picks.

## Multi-leader replication

Several nodes accept writes and replicate to each other (also called
active/active or bidirectional). Each leader is a follower of the
others.

**Synchronous multi-leader** collapses to “there is still one place
that must be reachable to commit.” The interesting case is
**asynchronous** multi-leader: each site keeps taking writes when
the WAN is down, and they merge later.

Inside **one** region this almost never pays for the complexity.
The usual honest reasons are **two (or more) datacenters that want
local writes**, and **devices that work offline**.

### Geographically distributed operation

Single global leader: every write from another continent pays a
round-trip to the leader’s region. Multi-leader: a leader per
region; intra-region you still do ordinary leader–follower (maybe
across AZs); between regions, leaders ship changes to each other.

You buy: write latency of the local region, and the ability to
write during a cross-region partition. You pay: **conflicts**
whenever the same row is updated in two regions before the change
arrives.

**Topologies.** Circular (A→B→C→A), star (one hub), all-to-all.
Sparse topologies have single points of failure in the *replication
path* — if the hub dies, edges stop converging. All-to-all can
deliver writes **out of causal order** (an `UPDATE` arrives before
the `INSERT` it depends on). Version vectors exist to detect that;
many products do not use them well. **Autoincrement keys**,
**uniqueness constraints**, and **triggers** become surprising
because there is no single allocator and no single “after this
statement” moment.

The book’s tone is: **avoid multi-leader unless you must.** If you
must, test failover and conflict paths; do not trust the brochure.

**Conflict avoidance** is the cheap strategy when it fits: pin each
user (or each record) to a **home region** so two leaders rarely
touch the same row. That is not multi-leader for that row anymore;
it is single-leader with a routing rule. Fine for “my profile,”
useless for a shared inbox two support agents edit at once.

### Sync engines and local-first (2nd edition)

Treat the **client device** as another replica. The user edits
offline; a **sync engine** captures local mutations, ships them when
the network exists, applies remote mutations, and updates the UI
without waiting for a server round-trip.

That is multi-leader with **millions of flaky leaders** and lag
measured in hours. Calendar apps, notes, field tools, and
collaborative canvases (several people typing without a lock) all
sit here. Real-time collaboration without offline is still
multi-leader: each browser tab is a replica that commits locally
and syncs asynchronously.

Jargon worth keeping straight:

- **Offline-first** — the app still works without a network.
- **Local-first** — stronger: it still works if *your* servers go
  away, typically because sync uses an open protocol and more than
  one provider (Git is the extreme, non-real-time example).
- **Sync engine** — the library that stores, sends, merges, and
  renders (CRDT stacks, Automerge, Yjs, Electric, Replicache-like
  tools, …).

Right model for some products. Wrong model for a **bank ledger**
or anything that needs a single global “this transfer happened
once.” Local-first also changes the privacy story: data lives on
the device (ch. 1 / 14).

### Dealing with conflicting writes

Two writes are **concurrent** if neither replica had seen the other
when it accepted the write. Wall-clock sameness does not matter;
offline edits hours apart are still concurrent.

Once you can detect that (version vectors — below), you must
**resolve**:

| Strategy | What happens | Typical failure |
|---|---|---|
| **Last-write-wins** (timestamp) | One value survives | Silent loss; clocks lie (ch. 9) |
| **Merge in the app** | You write the combinator | You forget a field; merge is wrong |
| **CRDTs** | Structure that always converges | Wrong CRDT for the domain (counters vs sets vs text) |
| **Siblings** | Store both, ask the user or a later job | UI and ops must handle twins |

There is no magic. If two leaders accepted different values for the
same key, the system must pick or combine. **Shopping carts** are
the canonical LWW disaster: two offline adds, LWW keeps one cart,
an item vanishes. A merge that **unions the items** is the actual
product.

Convergence (“eventually the same bytes”) is weaker than “we did
what the user meant.” Aim the conflict policy at the **invariant**,
not at the storage engine’s default.

## Leaderless replication

Dynamo, Cassandra, Riak, DynamoDB-ish designs: there is no failover
to wait for. The client (or a coordinator node) **writes to several
replicas**, **reads from several**, and uses **quorums**.

Parameters: `N` copies of a key, write waits for `W`, read waits for
`R`. If `W + R > N`, the read set and write set **overlap**, so a
read should see at least one replica that took the write — **until**
clocks, sloppy quorums, concurrent writes, and “the overlapping
node was the stale one and you picked by timestamp” enter.

Common default: `N=3, W=2, R=2`. You can still write if one node is
down. You cannot write if two are down, unless you relax the
quorum.

### Writing to the database when a node is down

**Hinted handoff:** if a replica is down, write the value to another
node with a hint; when the owner returns, the hint is forwarded.
**Read repair:** a read that sees mixed versions writes the newest
back to the stale replicas. **Anti-entropy:** background sync
(often **Merkle trees**) so replicas that missed a hinted handoff
still converge.

**Sloppy quorum** (Dynamo/Riak; Cassandra `ANY`): accept a write on
*whoever is reachable*, not necessarily the N “home” replicas. You
keep taking writes during a large partition; you **do not** have
the W+R overlap guarantee anymore. Subsequent reads can miss the
value. Use it only when “accept the write” beats “see it on the
next read.”

### Performance versus single-leader

You skip **leader failover pauses**. You pay coordination on *every*
write (fan-out to N, wait for W) and you get weaker consistency
unless you are very careful. Tail latency follows the *slowest* of
the W or R replies you wait for, so bumping W and R “for safety”
makes p99 worse. In practice quorums stay small (e.g. 3 of 5, 4 of
7).

Quorum reads and writes **alone are not linearizable**. Overlapping
majorities do not by themselves give you a total order clients can
rely on (ch. 10). A value can appear and then vanish on the next
read. Concurrent writes need a merge rule, not just `W+R>N`.

### Multi-region operation

Leaderless can put replicas in several regions and let `W`/`R` stay
in the local region for latency — which means **those local quorums
do not overlap with the far region**. You have reinvented async
multi-leader with extra steps. Name it: either you wait for a
cross-region replica (latency) or you accept regional divergence
and conflict resolution.

### Detecting concurrent writes

**Last-write-wins by timestamp** is still the default in too many
clusters. It is easy and it **drops data** whenever clocks are
wrong or two writes are genuinely concurrent.

A **version number** per key is not enough once there are several
replicas: replica A can be at v3 and replica B at v3 from a
*different* parent. You need a **version vector** (a counter per
replica, or an equivalent): it tells you whether write X
**happens-before** write Y, or they are concurrent. Concurrent
values are **siblings** until something merges them.

Merging siblings is application work — the same table as
multi-leader conflict resolution. Leaderless does not free you
from it; it makes it the normal case.

## How this shows up when you design something

Default sketch for a user-facing OLTP store:

1. Single-leader in one region, async replica for failover / reads.
2. Say what happens on failover (**RPO / RTO**), and that the old
   leader is **fenced**.
3. Say which reads may be stale, and how you keep **read-your-writes**
   for the session that just wrote (leader reads, LSN wait, or
   stickiness).
4. If you need multi-region writes, *name the conflicts* and the
   merge. If you cannot name them, you do not need multi-leader.
5. If you pick Cassandra-style, *name `N, W, R`*, whether quorums
   are strict or sloppy, and what stale means on the read path.
6. If the client is a replica (notes, docs, field app), you are in
   **sync-engine** land: CRDT/merge is the product, not an ops
   afterthought.

Do not draw three boxes labeled “replica” without saying who takes
writes.

## Check yourself

1. You acknowledged a write with async replication and then the
   leader died. What might a client have seen that is now gone?
   What number on a design doc names that loss?
2. Give a read-your-writes bug in a social app, and one fix that
   does not require linearizability everywhere.
3. Why is last-write-wins dangerous for a shopping cart? What would
   you merge instead?
4. When would you actually want multi-leader, and what will you do
   with conflicts?
5. A follower has been down for two days. The leader’s replication
   log only retains 24 hours. What happens when the follower comes
   back, and what is the operational fix *before* the next outage?
6. Two nodes both think they are the leader. What did you fail to
   do, and what is the usual token that prevents it?
7. `N=3, W=2, R=2` and one node is down. Can you write? Can you
   still lose a write the client thought succeeded? Why is this
   still not linearizable?
8. A collaborative notes app works on a plane. Which replication
   style is that, and why is it a bad fit for a payments ledger?

Continue to [Sharding](../7-sharding/).

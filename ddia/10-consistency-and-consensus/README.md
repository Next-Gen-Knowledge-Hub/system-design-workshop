# 10. Consistency and consensus

These notes teach consistency and consensus from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini. The second edition largely rewrote this material.

## What “the” value means

Several copies of the data sit on nodes that crash, pause, and lose messages. “The” value is a claim that those copies behave like one careful computer: a write finishes, and every read that starts after that finish sees the write. **Linearizability** is the precise name for that claim. **Consensus** — Raft, Paxos, Zab, and their relatives — is the tool that implements it on a cluster that can lose nodes. Done carefully, consensus is single-leader replication with automatic failover that does not lose a committed entry and does not elect two leaders.

“Strong consistency” is a brochure phrase. Many databases that look replicated are not linearizable. The difference is the whole subject.

## Linearizability

### One point on a timeline

A system is **linearizable** if it behaves as if there were one copy of the data, and every operation takes effect atomically at a single point in time between its invocation and its response.

Once a write of `x = 1` completes, any read that starts later must see `1`, or a later value. The answer does not depend on which replica received the read.

You can draw it. Each operation is an interval from invocation to response. Linearizability places a point inside each interval, and those points respect real-time order: if operation A’s response happens before operation B’s invocation, A’s point comes first.

That is a **recency** guarantee for a single object, often called a linearizable register, or for a compare-and-set on that object. It is a stricter demand than three neighbors people confuse with it:

- **Serializability** is about transactions. The outcome matches some serial order of those transactions. By itself it says nothing about replicas, and nothing about real-time order across clients. A serial order is allowed to put a transaction “before” another that actually finished earlier on the wall clock.
- **Causal consistency**, **sequential consistency**, and **eventual consistency** are weaker stories. They are what many geo-distributed systems actually provide: a reader may lag, as long as the lag respects a chosen rule (or, for eventual consistency, as long as replicas converge if writes stop).
- A quorum with `W + R > N` — the write waited for `W` replicas, the read waited for `R`, and those sets must overlap because their sizes sum to more than the number of copies — helps freshness. Overlap alone does not give linearizability under concurrency and timing. The overlapping replica can be the stale one, and a timestamp rule can prefer an older value.

Worked example of the guarantee breaking. A linearizable write of `x = 1` completes. A read starts after that completion and returns `0`. The recency guarantee failed. Somewhere a replica that had not applied the write was allowed to answer as if it held the current value.

### When the product needs it

Reach for linearizability when the product needs a single truth *now*:

- Uniqueness: “this username is taken,” decided at the moment of the write, visible to the next attempt.
- Locks, leases, and leader election.
- A single-object compare-and-set.
- Cross-client “read the thing that just finished writing,” where the reader is a different client from the writer and cannot rely on sticking to the same session.
- Some identifier-allocation stories, below.

A social feed, “people also bought,” and most analytics do not need it. Claiming linearizability for the whole graph is how a design becomes slow and fragile. “Read your writes” for one user is often enough with a sticky session or a read from the primary. That is a session guarantee, not linearizability of every object.

### How a replica set gets there

A single-threaded store on one node is linearizable until that node dies. A replica set is linearizable only if the protocol cannot serve stale data as “the” answer.

The pieces that do that:

- Writes commit only when a quorum has them, or when they have been synchronously replicated to a majority.
- Reads cannot return a stale answer from a former leader. A **leader lease** (the leader may answer reads only while a time-bounded promise says it is still the leader), a **quorum read** (a read waits for a majority and picks the latest), or a carefully proven read-your-writes path, are the usual tools. A casual “read from a follower” is not one of them.
- Asynchronous primary-and-replica replication is not linearizable. The primary can acknowledge a write the replica has not seen, and a read can land on that replica.
- A Dynamo-style quorum is not automatically linearizable. Overlapping read and write sets still allow a value to appear and then disappear, when a later read overlaps a different replica and resolves versions by a rule that can prefer the older one.

Consensus, below, is the usual way to get a linearizable replicated log. A **state machine** on top of that log applies the entries in the committed order, so the state inherits the log’s order.

### What it costs

If the network between replicas is slow, every linearizable operation waits for the protocol. In a geo deployment that wait can be tens of milliseconds, or it can be seconds.

During a network partition, a linearizable system refuses some operations — usually writes, sometimes reads — rather than return a guess. Systems that accept writes in both partitions are not linearizable for those writes. ZooKeeper and etcd stay available for the members that still have a quorum. The minority partition blocks. That is the trade.

A linearizable register, faced with writes on the minority side, must refuse those writes. If the minority committed them, a reader on the majority side could see a different value, and the “one copy” story would be false.

## Identifiers and logical clocks

`AUTOINCREMENT` on one node is a linearizable allocator and a single point of failure. Distributed identifiers trade properties off:

| Approach | Unique? | Ordered? | Linearizable? | What you give up |
|---|---|---|---|---|
| UUID v4 | yes | no | no | Safe and opaque. A poor clustered primary key, because inserts land at random places in a tree index. |
| Snowflake-style | yes | mostly by time | no | Depends on the clock. Includes a worker id so two nodes do not collide. |
| Pre-allocated blocks | yes | per allocator | no | Fast. A crash leaves gaps where a block was reserved and not fully used. |
| Consensus, or a single allocator | yes | yes | can be | Correct, and a bottleneck, because every id waits on that one agreement. |

### Clocks that track causality

A **Lamport clock** is a counter that rides along with messages. When you send, you attach your counter. When you receive, you set your counter to one more than the maximum of your counter and the sender’s. A **version vector** keeps one counter per replica, so you can tell “A happened before B” from “A and B were concurrent.” A **hybrid logical clock** mixes a wall-clock timestamp with a counter so the numbers stay close to real time while still respecting message order.

All three give an order that respects **happens-before**: if A could have affected B, A’s clock is less than B’s. None of them, by itself, gives linearizable real-time identifiers. They are the right tool for detecting concurrency and for causal consistency.

Snowflake-style ids are unique, and they are roughly ordered by time, and they are still not a linearizable id generator. Two workers can mint ids out of real-time order when their clocks disagree, and nothing in the id scheme waits for a quorum before the id is handed out. If the product needs “this id was assigned before that one, in real time, across the cluster,” you are back to consensus or a single allocator: a linearizable id generator.

## Consensus

Informally, consensus is a group of nodes agreeing on a value, or on a sequence of values, such that they will not change their minds, even if some nodes crash and messages are duplicated. Safety holds across a partition: they do not commit two different values for the same slot. Liveness holds when a majority can talk: a proposed value eventually commits.

### Problems that are consensus in disguise

These problems reduce to one another if you are careful about the reduction:

- Agreeing on the next log entry. That problem is **total order broadcast**: every surviving node delivers the same commands in the same order. A replicated log is the usual packaging.
- A linearizable compare-and-set.
- Leader election, and locks or leases: who owns this resource.
- Uniqueness under concurrency: two clients trying to claim the same username.
- **Atomic commit**: every shard says yes, or every shard says no. Two-phase commit is the classic protocol for that vote. Two-phase commit alone is not full consensus when the coordinator is a single point of failure — participants can be stuck in doubt if that one coordinator’s decision log dies. Serious systems combine the vote with a consensus group so the decision itself is replicated.

**Raft** and **Multi-Paxos** implement a replicated log:

1. A client proposes a command to the leader.
2. The leader appends the command as a log entry and sends it to the followers.
3. The entry is committed once a majority has persisted it.
4. Followers apply committed entries in log order.

Kafka’s controller epoch, etcd, ZooKeeper (which uses Zab), Consul, and the commit path of many NewSQL stores sit on this pattern. Once you have the log, you run a **replicated state machine**: the log is the truth, and the state is derived by applying the log. Two nodes that have applied the same prefix of the log are in the same state.

### A small cluster, on purpose

You run consensus on a small cluster, typically 3 or 5 voters, not on 50, and not on 500. Each extra voter is another replica that must persist the log before a majority exists, and the protocol’s failure tolerance is “a minority may be unreachable and the majority can still commit.” A voter set of 50 or 500 mostly adds latency and operational surface. The hundreds of machines that store user data get decisions by replication, by gossip, or by reading the small core’s output.

That small cluster is the **consistent core** for membership, for who the leader is, for configuration, and for shard maps. It is not the place you put petabytes of user rows.

What a Raft-style failover actually buys:

- No split-brain leaders for the same term or epoch. A term is the monotonically increasing generation of an election. Writes from an old term are rejected.
- No loss of entries that a majority had committed, after failover. An uncommitted tail that only the old leader had may be discarded; a committed entry may not.
- Automatic leader election when the old leader is partitioned away.

What it does not buy: Byzantine tolerance, in the usual deployment (nodes may crash, they are not assumed to forge messages); serializable multi-key transactions (that needs a transaction protocol on top of the log); or “a timeout means the command did not apply.” The client that timed out still does not know whether the majority committed the entry. The command has to be idempotent.

A fencing token is the same generation idea, applied after a pause. The old leader wakes up still believing it leads. Storage, or the new leader’s term check, rejects its writes because the token is from the previous term. Leader election without that check is how a paused process becomes a second writer.

### Coordination services

ZooKeeper, etcd, and Consul are small, strongly consistent stores optimized for coordination, not for bulk data.

The primitives they usually offer:

- Linearizable reads and writes of small keys.
- **Ephemeral** nodes, or leases, for membership: the node disappears if the session that created it dies, so “who is in the cluster” is a list that cleans itself up.
- **Watches**, notify-on-change, so a client observes derived cluster state instead of polling blindly.
- Leader-election recipes built on those primitives.

Use them as the core for “who owns this shard.” Do not use them as the user database. Putting high-QPS session traffic through etcd is how you find its throughput ceiling.

A job that must have only one owner takes a lease from that core, with a fencing token storage will check. A unique email at global query rate takes a linearizable index or an allocation service. The other product choice is eventual uniqueness: accept the write, and if a conflict appears later, tell the user to pick another address. A feed can stay eventual. The split is the design: this constraint is linearizable, this feed is eventual, and the split is acceptable because a stale feed entry does not sell the same username twice.

## Mechanism recap

- Linearizability places every operation at one point between its invocation and its response, on a single logical copy. After a write of `x = 1` completes, a read that starts later sees `1` or something newer. Serializability orders transactions; it does not by itself order replicas in real time. Causal, sequential, and eventual consistency, and a bare `W + R > N` quorum, are weaker.
- A replica set is linearizable when writes commit on a majority and reads cannot be answered by a stale ex-leader: a leader lease, a quorum read, or a proved read-your-writes path. Asynchronous replication and Dynamo-style quorums do not give you that by default. During a partition the minority refuses writes.
- UUID v4, Snowflake-style ids, and pre-allocated blocks buy uniqueness without linearizable real-time order. Lamport clocks, version vectors, and hybrid logical clocks respect happens-before and do not assign linearizable ids. A single allocator or a consensus log does.
- Raft and Multi-Paxos commit a log entry once a majority persists it, then apply a state machine in that order. Run 3 or 5 voters, not 50 or 500. The core decides membership, leadership, config, and shard maps. Failover keeps one leader per term and keeps majority-committed entries. A timeout still does not prove the command was lost.
- ZooKeeper, etcd, and Consul offer linearizable small keys, ephemeral membership, and watches. A lease without a fencing token does not stop a paused leader. User traffic and petabytes of rows do not belong on the core.

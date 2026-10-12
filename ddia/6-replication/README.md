# 6. Replication

These notes teach replication from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What replication is

Replication puts the same data on several nodes. One copy fails in ordinary ways: a disk dies, a zone dies, the people reading the data live on another continent, or the read rate outgrows one machine. The idea is a copy. The failures are the gaps between the copy that accepted a write and the copies that have not replayed it yet.

A set of copies is a mechanism, not a finished promise of high availability. What you actually get depends on who is allowed to accept writes, and on whether that node waits for another copy before it tells the client the write is committed.

People turn replication on for five jobs. Each job still has a failure the copies do not remove:

1. **High availability.** A surviving node can keep serving when a node, a zone, or a region dies. A bad deploy of the same binary still takes every replica down together, because they all run that binary.
2. **Durability.** A copy on another machine survives a machine that never comes back. `DROP TABLE` is itself a change in the log, so every replica replays it. The extra copy does not undo that statement.
3. **Latency.** A copy near the user makes a read local. A write whose leader is on another continent still pays the round trip to that leader.
4. **Read scale.** Followers can serve reads. Every write still has to reach every copy, so more followers do not raise write throughput.
5. **Disconnected and local-first work.** A device keeps editing while the network is gone. Two devices that edit the same record before they see each other still have to merge.

Older documentation calls the roles master and slave. The same roles are leader and follower.

Every style copies a **log of changes** and replays it. The styles differ in three answers:

1. How many nodes may append to that log.
2. Whether the writer waits until a copy is durable (**synchronous**) or acknowledges after its own persist (**asynchronous**).
3. What the system does when two writers accepted different values.

**Sharding** is the other split of a dataset: different records live on different groups of nodes. A production cluster almost always does both. Each shard — one slice of the keyspace — is its own small replicated group, with its own log and its own leader.

## Single-leader replication

One node is the **leader** (also called the primary). It is the only node allowed to append a write to the log. The other nodes are **followers** (replicas, secondaries). They receive the stream of changes and replay it in the same order.

A read can go to the leader, which has every write it has acknowledged, and which then becomes the bottleneck. Or a read can go to a follower, which spreads the read load and may not yet have the latest entries. That gap is **replication lag**.

This is the shape of a Postgres streaming replica, MySQL replication, a MongoDB replica set, and a Kafka partition leader. It is the usual production default because there is one order of writes: the order the leader appended.

When the leader dies, managed products usually fail over on their own. The mechanism under that button is still two hard steps: decide that the leader is dead (a timeout, which can be wrong), and agree which follower is the new leader (a consensus decision among the survivors, so two nodes do not both start accepting writes).

### Synchronous and asynchronous replication

**Synchronous replication** runs like this:

1. The client sends a write to the leader.
2. The leader appends it and ships it to a follower, or to a configured set of followers.
3. The leader waits until that follower has the write on durable storage.
4. Only then does the leader tell the client the write is **committed**, and only then does it usually show that write to other readers.

If the leader dies after that acknowledgement, a follower already has the write. The cost is that write latency includes the replica’s disk and the network hop to it. If that replica is slow or dead, writes stall, unless the operator drops it out of the synchronous set so a different replica can take its place.

**Asynchronous replication** runs like this:

1. The client sends a write to the leader.
2. The leader persists the write on its own disk.
3. The leader acknowledges the client immediately.
4. Followers receive the change later, whenever they can.

The acknowledgement is fast, and a dead follower does not stop writes. Failover can lose the last acknowledged writes: the client saw success, and the new leader’s log never contained the record. Lag has no bound. On a quiet local network it is milliseconds. Under load, after a replica restart, or across a bad wide-area link, it is minutes or hours.

**Semi-synchronous** replication, also called “wait for N,” sits between those two. The leader waits for at least one follower — or for `min.insync.replicas` followers — and does not wait for every follower. Kafka’s `acks=all` together with a minimum in-sync replica count is this idea applied to a log: the leader acknowledges once the required followers have the record, and a single slow replica outside that set does not stall the write.

Two numbers name the leftover risk. **RPO** (recovery point objective) is how much acknowledged data a failover is allowed to lose. **RTO** (recovery time objective) is how long promotion of a new leader is allowed to take.

A two-region case shows what synchronous really buys. The leader is in region A. One follower is synchronous and lives in region B. The network between the regions dies. The leader cannot get the durable acknowledgement from B, so it cannot commit. That arrangement is still single-leader replication with a remote veto. It is not two regions both accepting writes.

### Setting up a new follower

Copying a live data directory with `rsync` and hoping is not a replica. The new node needs a **consistent snapshot** — a frozen picture of the data at one moment — plus a position in the replication log, so it can replay exactly the changes that happened after that picture.

1. Take a snapshot of the leader, or of a follower that has already caught up, at a known log position: a log sequence number, an **LSN** (the byte offset Postgres uses in its write-ahead log), or a binlog coordinate (MySQL’s name for the same kind of position).
2. Copy that snapshot to the new node.
3. The new follower connects and asks for every change after that position, applies the backlog, and then joins the live stream.

If the leader has already discarded the log past that position, the new node cannot catch up. The only fix is a newer snapshot and a position that still exists in the retained log. **Log retention** — how long the leader keeps old log — is the knob that decides whether a new node can join. Without snapshot plus catch-up, the cluster cannot grow, a dead disk cannot be replaced, and a replica cannot be cloned for analytics.

A follower that has been down for two days, against a leader that retains only 24 hours of log, comes back to a gap the leader can no longer fill. It does not “resume.” It is rebuilt from a new snapshot taken while the log still covers the snapshot’s position. The operational fix, before the next outage, is retention long enough for the longest outage you intend to catch up from, or an explicit rule that an absent follower is rebuilt from backup instead of catching up.

### When a node is down

**A follower dies, or is cut off by a network partition** (a broken link, not a slice of the keyspace). It already has a local copy of every change it applied before the break. When it returns it asks the leader for the missing range and applies it. The idea is simple. The I/O is not: if the follower was down for a long time, or the write rate is high, both the leader (reading and shipping the backlog) and the recovering follower (applying it) spend their disks on catch-up.

The leader wants to delete old log once every follower has it. An absent follower forces a choice. Keep the log, and the leader’s disk fills. Drop the log, and that follower cannot catch up and must be rebuilt from backup. A Postgres **replication slot** is that contract made explicit: the slot pins the log until the subscriber named by the slot has consumed it, so “delete” and “this follower still needs it” are one decision instead of an accident.

**The leader dies.** Failover is four steps:

1. **Detect** death with a timeout. A long timeout makes users wait on a leader that is already gone. A short timeout treats a garbage-collection pause or a network blip as death. The needless failover that follows makes a sick cluster sicker, because promotion, client redirection, and catch-up all add load.
2. **Elect** a new leader. With asynchronous replication, the candidate is the follower with the highest log sequence. Promoting a replica that is days behind silently throws away every write the old leader had acknowledged and that this replica never received.
3. **Reconfigure** clients and the remaining followers so they send writes and replication traffic to the new leader.
4. **Fence** the old leader. Fencing stops it from accepting writes if it comes back. The usual tool is a monotonically increasing **generation**, also called a **term** or an **epoch**: a number that goes up on every successful election. Storage rejects writes that carry an old generation. Without fencing, two leaders accept writes, their logs diverge, and someone merges them by hand — or nobody notices until checksums disagree.

Some teams still fail over by hand even when the software can do it automatically. A false failover under load costs more than a few extra minutes of unavailability. Automatic failover is a policy with its own failure modes, not free high availability.

### How the replication log is encoded

The bytes on the stream matter as much as which node leads.

**Statement-based** replication ships the SQL text (`UPDATE …`) and replays the statement on the follower. It breaks on `now()`, `rand()`, triggers, row-by-row differences, and non-deterministic user-defined functions, because the follower executes the statement again and can get a different result.

**Write-ahead log (WAL) shipping**, also called physical replication, ships the redo bytes the storage engine already wrote. Postgres streaming replication is this. It breaks when the engine or the major version does not match, and the bytes are hard to decode as events another system could read.

**Logical, or row-based, replication** ships a change event: this row became this value. It costs a bit more CPU, and the consumer still has to order the events and filter them. Logical decoding is what lets a system replicate a subset of tables, or feed **change data capture (CDC)** — a stream of row changes — into Kafka.

Statement-based replication looks simple and is a museum of edge cases. Treat it as a historical warning.

**Trigger-based replication** is older still: a trigger writes a row into a change table, and something else tails that table. It is an application-level log with extra failure modes (the trigger and the change table can diverge from the engine’s own log). Prefer the engine’s logical stream when the engine has one.

### What replication lag does to a session

Reading from a follower is a performance feature with a consistency cost. Three session bugs show up on a healthy cluster. None of them needs a full network partition. Two hundred milliseconds of lag is enough.

**Read-your-writes**, also called read-after-write, fails like this:

1. You submit a post. The leader acknowledges it.
2. The app redirects you to the post page.
3. That page reads a follower that has not replayed the insert.
4. The page is empty. You think the submit failed and you click again.

**Monotonic reads** fail like this:

1. You load your profile and see the new bio. That read hit a follower that had caught up.
2. You refresh. The load balancer sends you to an older follower.
3. The bio is the previous one. Time went backward for one user.

People report “the site is haunted.” They do not report “eventual consistency.”

**Consistent prefix** fails when a reply is visible before the message it replies to. Those two rows live on different shards, and each shard’s followers lag by a different amount, so the reply’s shard has applied its write and the message’s shard has not. Inside one shard, a single leader applies writes in one order, so a reply cannot become visible before the message it depends on *on that shard*. Across independent shards there is no shared order unless the system carries **causal metadata**: information that says “this write happened after that write,” so a reader can wait until both are visible.

Do not design for the lag on the dashboard at noon. Design for the lag you see while a replica is rebuilding.

Three application mitigations cover the common cases. They are easy to get wrong.

- After a user writes, read **that user’s** writes from the leader, or from a synchronous replica. **Session stickiness** is the blunt form: pin that session to the leader for a few seconds after the write.
- **Read at a timestamp or LSN.** The client remembers the log position of its write and waits until the follower’s applied position has passed it. Bitbucket and others have published this pattern.
- Put causally related rows on the **same shard**, so they share one log order and a reply cannot outrun its parent.

A database that offers **linearizability** — once a write completes, any later read sees that write or something newer, no matter which replica answers — together with transactions across the replicas you read, makes those mitigations unnecessary. That is the NewSQL shape: CockroachDB, Spanner-like systems, TiDB. You pay a coordination round on the commit, and you still have to understand which region that round crosses.

Weaker replication remains a valid choice when the system must keep accepting writes through a network blip, or when a consensus round on every commit is more latency than the product can spend. The rest of this lesson is those choices.

## Multi-leader replication

In **multi-leader** replication, also called active/active or bidirectional replication, several nodes accept writes. Each leader ships its log to the others, and each leader is a follower of the others.

Synchronous multi-leader collapses: a commit still needs every participating leader to be reachable, so there is still one set of nodes that must answer before the write exists. The interesting case is asynchronous multi-leader. Each site keeps taking writes while the wide-area network is down, and the sites merge when the link returns.

Inside one region the extra machinery almost never pays for itself. The honest reasons are two or more datacenters that want local writes, and devices that must work offline.

### Writes in more than one region

With one global leader, a write from another continent pays a round trip to the leader’s region before the client hears “committed.”

With a leader per region, the path splits:

1. Inside a region, ordinary leader–follower replication still runs, often across availability zones.
2. A client in that region writes to the local leader and gets a local-region acknowledgement.
3. Between regions, the leaders ship changes to each other asynchronously.

You buy the write latency of the local region, and you can write during a cross-region partition. You pay a **conflict** whenever the same row is updated in two regions before either change has arrived at the other.

Three **topologies** describe who ships to whom:

- **Circular:** A ships to B, B to C, C to A.
- **Star:** every region ships to one hub, and the hub ships back out.
- **All-to-all:** every leader ships to every other leader.

A sparse topology has a single point of failure on the *replication path*. If the hub dies, the edges stop converging even though each edge can still accept local writes. All-to-all can deliver writes out of causal order: an `UPDATE` arrives at a replica before the `INSERT` it depends on, because the two statements traveled different paths. **Version vectors** — a counter per replica, described below — detect that the update and the insert were not in happens-before order. Many products implement that detection poorly.

**Autoincrement keys**, **uniqueness constraints**, and **triggers** become surprising because there is no single allocator and no single moment of “after this statement, on the one leader.” Two regions can mint the same numeric id, or both accept a username the other region has also accepted.

Avoid multi-leader unless a site must accept writes while it cannot reach the others. If you use it, test failover and the conflict path. A brochure that says “active/active” has not run those tests for you.

**Conflict avoidance** is the cheap strategy when it fits the data. Pin each user, or each record, to a **home region**, and route that user’s writes to the one leader there. For that row, the system is single-leader again, plus a routing rule. That works for “my profile.” It does nothing for a shared inbox that two support agents edit at the same time in two regions.

### Sync engines and local-first software

Treat the user’s device as another replica. The user edits while offline. A **sync engine** captures the local mutations, ships them when a network exists, applies mutations that arrived from elsewhere, and updates the screen without waiting for a server round trip.

That is multi-leader replication with millions of flaky leaders, and with lag measured in hours. Calendar apps, notes, field tools, and collaborative canvases — several people typing in one document without a lock — sit here. Real-time collaboration that does not even offer offline mode is still multi-leader: each browser tab is a replica that commits locally and syncs asynchronously.

Three names are easy to collapse, and they are not the same:

- **Offline-first** means the app still works with no network.
- **Local-first** is stronger: the app still works if *your* servers go away, typically because sync uses an open protocol and more than one provider. Git is the extreme example, and it is not real-time.
- A **sync engine** is the library that stores, sends, merges, and renders. CRDT stacks, Automerge, Yjs, Electric, and Replicache-like tools are examples. A **CRDT** (conflict-free replicated data type) is a data structure whose merge rule always converges, covered with the other merge rules below.

This is the right model for some products. It is the wrong model for a bank ledger, or for anything that needs one global fact of the form “this transfer happened once.” Local-first also changes the privacy story: the data lives on the device.

### Conflicting writes

Two writes are **concurrent** when neither replica had seen the other at the moment it accepted its write. Equal wall-clock timestamps do not matter. Two offline edits hours apart are still concurrent if neither device had received the other’s edit.

Once version vectors show that the writes are concurrent, something must **resolve** them:

| Strategy | What happens | How it fails |
|---|---|---|
| **Last-write-wins** (a timestamp picks one value) | One value survives | The other value is dropped with no error. Clocks are wrong often enough that the “last” write is not the write that happened last. |
| **Merge in the application** | You write the function that combines the two values | You forget a field, and the merge is silently wrong. |
| **CRDTs** | The data structure’s own rule always converges to one value | The CRDT does not match the domain: a counter, a set, and a text document merge differently, and the wrong one preserves the wrong information. |
| **Siblings** | Both values are stored, and a user or a later job picks | The interface and the operators have to handle twins until that pick happens. |

If two leaders accepted different values for the same key, the system picks or combines. There is no third option hiding in the replication protocol.

A shopping cart is the canonical last-write-wins failure. Two devices each add an item while offline. Last-write-wins keeps one cart. The other item vanishes. The merge the product actually wants unions the items.

**Convergence** means the replicas eventually hold the same bytes. That is a weaker promise than “we did what the user meant.” Aim the conflict policy at the invariant — the cart contains every item someone added — rather than at whatever default the storage engine ships.

## Leaderless replication

Dynamo, Cassandra, Riak, and DynamoDB-style designs have no leader election to wait for. The client, or a coordinator node in front of the client, writes to several replicas and reads from several replicas. The rule that decides how many must answer is a **quorum**.

Three parameters describe one key:

- `N` is how many copies of the key exist.
- `W` is how many replicas must accept a write before the coordinator acknowledges it.
- `R` is how many replicas a read waits for.

If `W + R > N`, the read set and the write set overlap, so a read should see at least one replica that took the write. That overlap holds until clocks, sloppy quorums, concurrent writes, or a bad timestamp rule get involved: the overlapping node can be the stale one, and the reader then picks a value by timestamp and throws the fresher value away.

A common default is `N = 3`, `W = 2`, `R = 2`. One node can be down and a write still succeeds. Two nodes down, and a write cannot complete, unless you relax the quorum and accept a weaker answer.

### Writing while a node is down

**Hinted handoff** runs when a replica that should own the key is down:

1. The coordinator writes the value to some other reachable node, with a hint that names the real owner.
2. When the owner returns, the hint is forwarded and the owner stores the value.

**Read repair** runs on the read path. A read sees mixed versions across the `R` replies. The coordinator writes the newest version back onto the stale replicas.

**Anti-entropy** is a background sync, often using **Merkle trees** (hash trees that let two nodes find which key ranges differ without shipping every key). It exists so a replica that missed a hinted handoff still converges later.

A **sloppy quorum** (Dynamo and Riak; Cassandra’s `ANY` acknowledgement) accepts the write on whoever is reachable, not necessarily on the `N` home replicas for that key. The cluster keeps taking writes during a large partition. The `W + R > N` overlap is gone, because `W` and `R` were counted on stand-in nodes. A later read can miss the value. Use a sloppy quorum only when accepting the write matters more than seeing it on the next read.

### How this compares with a single leader

Leaderless replication skips the pause of leader failover. Every write pays coordination instead: fan the write out to `N` replicas and wait for `W` replies. Consistency is weaker unless the read and write paths are built carefully on top of that quorum.

Tail latency follows the slowest of the `W` or `R` replies you wait for. Raising `W` and `R` “for safety” makes the 99th percentile (p99) slower. In practice the quorums stay small: 3 of 5, or 4 of 7.

Quorum reads and writes by themselves are not linearizable. Overlapping majorities do not give clients a total order they can rely on. A value can appear on one read and vanish on the next: the second read overlapped a different, staler replica and picked by timestamp. Concurrent writes need a merge rule. `W + R > N` is not that rule.

### More than one region

A leaderless cluster can place replicas in several regions and let `W` and `R` be satisfied inside the local region, for latency. Those local quorums do not overlap the far region’s replicas. The result is asynchronous multi-leader replication with extra steps. Name the choice: either a write waits for a cross-region replica, and you pay the latency, or the regions diverge and you run conflict resolution.

### Detecting concurrent writes

Last-write-wins by timestamp is still the default in too many clusters. It is easy to operate, and it drops data whenever clocks are wrong or two writes are genuinely concurrent.

A single version number per key is not enough once several replicas accept writes. Replica A can be at version 3 and replica B at version 3, from different parents. Neither number says which write the other has seen.

A **version vector** is a counter per replica (or an equivalent structure). Compare two vectors:

- If every counter in X is less than or equal to the same counter in Y, and at least one is strictly less, then X **happens-before** Y. Y already includes X.
- If some counters are greater in X and others are greater in Y, the writes are concurrent.

Concurrent values are **siblings** until something merges them. That merge is application work, the same table as multi-leader conflict resolution. Leaderless replication does not remove the merge. It makes the merge the normal case, because any replica in the quorum can accept a write.

## Mechanism recap

- A leader appends writes to one log; followers replay that log. Synchronous replication waits for a follower’s durable copy before acknowledging; asynchronous replication acknowledges on the leader’s own persist and can lose that acknowledgement in failover.
- A new follower joins from a consistent snapshot plus a log position (LSN or binlog coordinate). If retention has already dropped that position, the follower is rebuilt, not caught up. An absent follower pins the log (a replication slot) or is sacrificed.
- Failover detects death by timeout, promotes the follower with the highest log sequence, redirects clients, and fences the old leader with a generation, term, or epoch. Statement-based, physical WAL, and logical row streams are three encodings of the same log, and they break in different places.
- Follower reads at a couple of hundred milliseconds of lag break read-your-writes, monotonic reads, and consistent prefix. Leader reads, an LSN wait, or same-shard placement patch a session; linearizable replication removes the patches by coordinating the commit.
- Multi-leader and leaderless replication (`N`, `W`, `R`, with hinted handoff, read repair, Merkle anti-entropy, and sloppy quorums) keep accepting writes through a partition and then merge concurrent values with version vectors, last-write-wins, an application merge, a CRDT, or siblings. A device running a sync engine is that same multi-leader pattern with the client as a replica.

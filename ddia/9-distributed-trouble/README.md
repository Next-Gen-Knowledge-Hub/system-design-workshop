# 9. The trouble with distributed systems

These notes teach partial failure from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What partial failure is

On one computer, `x = 1` either happened or the process died. On two computers, **partial failure** is normal. One node did the thing. The other never heard. The packet is sitting in a queue. Or the reply will arrive in 30 seconds. Distributed system design is tolerating that, without pretending it is rare.

The same demons show up in every protocol: a crash, a delay, a pause, and clocks that do not agree. Replication, sharding, and transactions all get their ugly edges from these, because each of those algorithms sends a message and then has to decide what silence means.

A **fault** is one component misbehaving. A **failure** is the system as a whole stopping the delivery of what it promised. The point of a distributed system is to contain faults so they do not become a service failure.

Partial failure is the defining property. Some nodes work, some do not, and the working ones cannot reliably know which is which. There is no shared memory, no global variable, and no operating-system call that returns the truth about the cluster. The only channel is the network, and the network is unreliable.

If the problem fits on one machine, that is often the better design. Distribution is a cost you pay for geographic latency, for isolating faults, and for scale. The rest of this lesson is that cost.

## Unreliable networks

### What TCP actually promises

TCP delivers a **reliable byte stream** if both ends stay up and the network eventually delivers the bytes. Inside one connection it also stops the application from seeing those bytes reordered.

It does not give you:

- A bound on how long a send or an acknowledgement takes.
- A way to tell “the peer crashed” from “the peer is slow” from “the network dropped my packets.”
- Protection from a switch that black-holes 1% of packets for an hour, or from a middlebox that resets long-lived connections.

“The request was sent” is not “the server processed it.” “No response” is not “it did not process it.”

### What actually breaks

Datacenter networks are usually excellent, until they are not. Real outages include asymmetric routing (A can send to B, B cannot send to A), silent packet loss, garbage-collection pauses that look like a stalled network, cloud network partitions between availability zones, and the classic case where ping works and the application port is black-holed. Wide-area links between regions are worse: higher latency, more variance, and more ways for politics and physics to cut the path.

Design as if any message may be lost, delayed, duplicated, or reordered. TCP prevents reordering inside one connection. Your logical retries create duplicates at the application layer: the client sends the request again because it did not see a response, and the server may already have done the work.

### Deciding that a node is dead

Most systems have no perfect failure detector. They use **heartbeats** (a periodic “I am alive”) and **timeouts** (silence longer than a guess means dead), sometimes with gossip-style suspicion, where nodes share “I have not heard from C” instead of each node deciding alone.

A node that answers slowly — a **limping** node — is harder than a node that is dead. It is alive enough to confuse detectors, because a heartbeat sometimes gets through, and dead enough to ruin service-level objectives (SLOs): the latency targets the service promised. Declare it dead, and you move its load onto everyone else. Leave it in the rotation, and it poisons the tail of every request that lands on it. A clean crash is the kinder fault: the timeout fires, the node is gone, and no slow replies come back.

### Timeouts

Timeouts are guesses. Too short, and you declare a live node dead, fail over, risk a split brain (two nodes both acting as leader), and amplify load with retries. Those retries can become a **metastable** storm: the extra load keeps the remaining nodes overloaded after the original fault is gone, so the system stays slow because of the retries themselves. Too long, and users wait on a node that will not answer. Adaptive timeouts — a percentile of recent round-trip times — track a network that got slower. They do not remove the ambiguity of a single silent call.

When a call times out, you are in one of three worlds:

1. The request never reached the peer.
2. The peer did the work, and the response was lost.
3. The peer is still doing the work, or will do it when it gets the request.

That is why idempotent APIs and dedupe keys exist. **Idempotent** means doing the operation twice leaves the same state as doing it once. A **dedupe key** is an id the server stores with the first successful attempt, so the retry is recognized and not applied again. Without them, “retry on timeout” doubles a charge.

Worked example. You send `POST /charge` and get no response. World 1: the card was never charged; a retry with the same key charges it once. World 2: the card was charged and the HTTP response died; a retry without a key charges it again. World 3: the charge is in progress; a retry that arrives while the first attempt is still running can charge it twice unless the server treats the key as a lock, not only as a record written after success. The API has to provide that key. The client cannot tell the three worlds apart from silence.

### Bounded delay, and the networks you actually have

A **synchronous** network model assumes delay is bounded: every message arrives within a known maximum, or it is declared lost. Hard real-time systems can buy something close, with dedicated hardware, low utilization, and a lot of money.

The public internet and a typical datacenter are **asynchronous**: delay is unbounded. A useful middle model is **partial synchrony**. For safety — nothing irrevocably wrong happens — the algorithm never needs the bound to be true. For liveness — a request eventually gets a response — the algorithm assumes that eventually there is a long enough stretch of stable, timely messages to make progress.

If a paper’s proof needs bounded delay for safety, be suspicious of that proof on cloud infrastructure. A bound you hope for is not a bound you have.

## Unreliable clocks

### Two clocks on one machine

A **time-of-day clock**, also called a wall clock, answers “what is the date?” It jumps when NTP steps the clock, when a virtual machine pauses and resumes, and when someone sets the time. It can go backwards. Across machines it differs by milliseconds to seconds, sometimes more.

Do not use a wall clock alone to order writes. Naive last-write-wins — keep the value whose timestamp is greater — will drop a write that happened later on a machine whose clock is behind. Do not use a wall clock alone to decide that a lock has expired, either. The holder and the waiter do not share a clock, so “expired” on one machine is not “expired” on the other.

A **monotonic clock** (`CLOCK_MONOTONIC` and its relatives) measures elapsed time on one machine. It does not jump backward when NTP steps the wall clock. It is the right tool for “this RPC took 2 seconds,” because that question is a duration on one process. It does not compare across machines. There is no shared zero. “Expires at 17:00 UTC” is a wall-clock question, and it is the dangerous one: 17:00 on this machine is not 17:00 on the lock server if the clocks disagree, so a lease can look valid to the holder after the rest of the cluster has already given it to someone else.

### How close synchronized clocks get

NTP is best-effort. It nudges clocks toward each other. It does not promise a maximum error.

Google **TrueTime**, and systems in the same family, expose an **uncertainty interval** instead of a perfect timestamp: the true time lies somewhere inside a window, and the API tells you how wide the window is. If you must use wall time for ordering or for commit visibility, treat that uncertainty as part of the API. **Wait out the error bound** before exposing a write — a clock-bound wait — so that every machine’s window has moved past the commit time and no reader can still consider the write to be in the future. Spanner-style systems do this. An application’s `Date.now()` does not. It returns a single number with no interval attached.

Bad uses of synchronized clocks: last-write-wins conflict resolution across regions; assuming timestamp order is causal order (A’s timestamp is smaller, so A happened before B); expiring leases with only a wall-clock time-to-live and no fencing token. A calendar is a concrete casualty of the first one. Two people edit the same event from two regions. The edit that the user made second carries the smaller timestamp, because that laptop’s clock is slow. Last-write-wins keeps the earlier edit and throws the later one away. The calendar looks consistent and is wrong.

Good uses: approximate event time for analytics, with a plan for data that arrives late; human-readable logs; a legal “as of” timestamp that states its precision.

### A process that pauses

Garbage collection, swapping, “the hypervisor froze my VM,” a `SIGSTOP`, a long stop-the-world pause, a container CPU throttle. The process wakes up holding a stale view. It still thinks it is leader. Its lease expired in real time while it was frozen. Its cache is ancient. Heartbeats did not go out, so peers declared it dead and elected someone else. Then the paused process writes anyway.

That is the **zombie** problem. “Pause less” is not a fix. You will never pause for zero time. The fix is **fencing**: a monotonic generation, or a token, that storage checks on every write, so a zombie’s write is rejected even if the zombie believes it still holds the lock.

The broken sequence, in order:

1. The client acquires a lock and reads “I hold it” into its own memory.
2. A garbage-collection pause freezes the client past the lease deadline.
3. The lock service grants the lock to someone else, who writes good data.
4. The paused client wakes up, still believes `lock.acquired`, and writes.
5. The write lands, because nothing at the storage layer checked a token. The new holder’s data is gone.

`if lock.acquired: write_db()` is that bug. The boolean lives in the client. Storage never saw it.

## Knowledge, truth, and lies

There is no shared memory. The only facts a node has are messages it received. It cannot know what it was not told. Protocols have to work from **local knowledge** only.

### A majority

A **majority quorum** is how you make a decision that a different partition of the cluster cannot also make. In a group of `2f + 1` nodes, two majorities both have to contain more than half the nodes, so they overlap in at least one node. That overlap cannot vote for two contradictory committed values at once. A minority that cannot reach a quorum must stop. It must not invent a new truth and start serving it.

That is the core of a “CP” system during a partition: it keeps its consistency promise and refuses to answer on the side that lacks a majority. It is also the core of Raft and Paxos, the consensus protocols that use that overlap to agree on a log.

### Locks and leases

`SETNX` in Redis, or a row in MySQL, is not a lock that survives pauses by itself. Two pieces have to sit on top:

- A **lease** is time-bounded and must be renewed, so a dead holder does not block the lock forever. When the time runs out, someone else may take it.
- A **fencing token** is a monotonic number from the lock service. Storage checks it on every write. A write with an older token is rejected. A zombie whose lease expired cannot overwrite the new holder’s work, because its token is smaller.

Without fencing, the client that paused still holds a lock in its own imagination. A lock that lives only in the client’s memory is theater.

### Nodes that lie

A **Byzantine** fault is a node that lies or returns garbage: a bug that forges a plausible message, malice, or a bit flip that still looks like a valid message. Most data systems assume **non-Byzantine** faults, either crash-stop (the node is gone) or crash-recovery (the node reboots with its disk intact). They assume a node does not send cryptographically plausible evil responses on purpose.

Checksums and disk ECC fight corruption of bits. Cryptography and blockchains spend their design on Byzantine agreement, where participants may lie. An ordinary operational database cluster usually does not pay that cost, except where you add verification on purpose: checksums on pages, signatures on an audit log.

### The model a paper assumes

When you read a paper, it assumes a **system model**, and the model has to say three things:

- What can fail: a crash, a dropped message, or a lie.
- Whether a node can recover with its disk intact.
- Whether delays are bounded.

**Safety** means nothing irrevocably wrong happened. Two leaders committing different values for the same log slot is a safety failure. **Liveness** means something good eventually happens: a request gets a response. Under a total partition you often keep safety and sacrifice liveness — refuse writes rather than guess.

That is the honest reading of CAP. During a partition you choose linearizable consistency (a single, recency-ordered value, and you refuse the side that cannot confirm it) or availability of conflicting writes (both sides accept writes, and you reconcile later). You do not get both for those writes.

### Models and randomized tests

**TLA+** and PlusCal check a specification by exploring its states. **Jepsen**-style fault injection kills nodes and drops packets against a real build. **Deterministic simulation**, the testing approach FoundationDB is known for, replaces the network, the clock, and the scheduler with a controlled fake so a rare interleaving can be replayed.

You will not write a TLA+ spec for every CRUD service. “We wrote a lock,” with no model and no chaos test, is hope. Deterministic simulation makes the interleavings you thought of reproducible. Hash iteration order, out-of-memory kills, and other nondeterminism still sneak back in. The point is vigilance. There is no single tool that closes the gap.

## Mechanism recap

- A fault is one component misbehaving. A failure is the service breaking its promise. Partial failure is normal: the only channel is an unreliable network, and a live node cannot ask a shared memory who else is alive.
- TCP gives a reliable, ordered byte stream while both ends and the network eventually deliver. It does not bound delay, and it does not tell a crash from a stall from a drop. Application retries duplicate work. A timeout leaves three worlds — never arrived, done but the reply died, still in progress — so the API needs an idempotency key.
- Timeouts and heartbeats are guesses. A short one causes needless failover, and the retries can metastable-storm the remaining nodes. A long one makes users wait. A limping node is worse than a clean crash because it stays in the rotation and misses the SLO. Adaptive timeouts track recent round trips and still leave one silent call ambiguous.
- Wall clocks jump, go backwards, and disagree across machines by milliseconds to seconds. Monotonic clocks measure elapsed time on one machine. TrueTime-style APIs expose an uncertainty interval and wait out the error bound. Last-write-wins and wall-clock lease expiry trust a clock they do not have.
- A paused process becomes a zombie leader. A lease expires the holder; a fencing token checked by storage rejects the zombie’s write. A majority of `2f + 1` overlaps any other majority, so a minority must stop. Safety stays; liveness waits. Byzantine lying is a different, more expensive model than crash-recovery.

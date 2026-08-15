# 9. The trouble with distributed systems

Companion notes for **Chapter 9** of *Designing Data-Intensive Applications*
(2nd ed.). Chapters 6–8 described algorithms. This chapter is why those
algorithms are ugly: the network lies, clocks lie, processes freeze, and
you cannot tell which.

The same four demons show up everywhere: crash, delay, pause, and
unsynchronized clocks. This chapter is how DDIA names them so you can
design for them on purpose.

## What this chapter is for

On one computer, `x = 1` either happened or your process died. On two
computers, **partial failure** is normal: one node did the thing, the
other never heard, the packet is in a queue, or you will get the
reply in 30 seconds. Distributed system design is **tolerating partial
failure** without pretending it is rare.

If you can keep the problem on one machine, chapter 1 already told you
to consider it. This chapter is the cost of the other choice.

## Unreliable networks

TCP delivers a **reliable stream** *if both ends stay up and the
network eventually delivers*. It does not give you:

- A bound on **how long** a send takes.
- A distinction between “peer crashed” and “network is slow.”
- Protection from the datacenter switch that drops 1% for an hour.

**Timeouts** are guesses. Too short: you declare a live node dead and
fail over (split brain risk). Too long: users wait. Adaptive timeouts
help; they do not solve the ambiguity.

**Fault detection** is therefore probabilistic. Gossip failure
detectors, heartbeats,
and “limping” nodes (alive but too slow to be useful) are all worse
than the textbook crash.

**Synchronous vs asynchronous networks:** a theoretically synchronous
network has bounded delay. The internet and most datacenters do not.
Distributed algorithms that assume bounds are assuming a fiction; the
useful ones work in **partial synchrony** (eventually, things are
stable enough).

## Unreliable clocks

**Time-of-day clocks** (wall clocks) jump when NTP steps, when a VM
pauses, when someone sets the date. They can go **backwards**. They
differ across machines by milliseconds to seconds (sometimes more).
Do not use them to order writes (last-write-wins) or to expire a lock
without extra care.

**Monotonic clocks** (`CLOCK_MONOTONIC`) are for measuring elapsed
time *on one machine*. They do not compare across machines.

**Clock sync:** NTP, Google TrueTime (bounded uncertainty, still not
zero). If you *must* use wall time (legal timestamps, TTL), treat
**uncertainty** as part of the API: wait out the error bound before
exposing a write (**clock-bound wait**: do not expose a result until the
uncertainty window has passed).

**Process pauses:** GC, swapping, “the cloud froze my VM,” a SIGSTOP.
The process wakes up with a stale view: it still thinks it is leader,
its lock TTL has expired, its cache is ancient. Heartbeats did not
go out, so peers declared it dead. Then it writes anyway. That is
the **zombie / fencing** problem (ch. 10, generation clocks).

## Knowledge, truth, and lies

There is no shared memory. The only facts you have are messages you
received. **Majority quorum** is how you make a decision that another
partition cannot also make (overlap of majorities). A minority that
cannot talk to anyone must **stop**, not invent a new truth.

**Distributed locks and leases:** “SETNX in Redis” is not a lock that
survives pauses. You need:

- A **lease** (time-bounded, must renew).
- A **fencing token** (monotonic number from the lock service) that
 storage checks, so a zombie with an expired lease cannot write.

Without fencing, the client that paused still holds a “lock” in its
own imagination.

**Byzantine faults:** nodes that lie or return garbage (bugs, malice,
bitrot). Most data systems assume **non-Byzantine** (crash-stop or
crash-recovery): you may die, you don’t encrypt evil responses. Crypto
and blockchain spend their lives here; your OLTP cluster usually does
not, except for checksums against corruption.

## System model and reality

When you read a paper, it assumes a **system model**:

- What can fail (crash, omit messages, lie)?
- Can nodes recover with disk intact?
- Are delays bounded?

**Safety** vs **liveness:** safety = nothing irrevocably wrong happened
(two leaders committed different values). Liveness = something good
*eventually* happens (a request gets a response). Under total network
partition, you often keep safety and sacrifice liveness (this is the
honest reading of CAP: during a partition you choose consistency or
availability of writes).

**Formal methods and randomized testing (2nd edition):** TLA+, Jepsen,
deterministic simulation (FoundationDB’s testing religion). You will
not TLA+ every service. You *should* know that “we wrote a lock”
without a model or a chaos test is hope, not engineering. Randomized
fault injection finds the pause/network cases humans do not storyboard.

## How this shows up when you design something

- Every arrow on the whiteboard needs a timeout *and* a retry policy
 that cannot metastable-storm (ch. 2).
- Every lock needs fencing.
- Every “the replica is healthy” is a timeout, not a fact.
- Every timestamp in a conflict resolver is a confession that you
 trust clocks.

In interviews, calling out “we cannot know if the payment RPC succeeded
if we timed out” is the adult move. Idempotent APIs (ch. 5 / 13) exist
because of this sentence.

## Check yourself

1. You sent `POST /charge` and got no response. What are the three
 worlds you might be in, and what must the API provide so retry is
 safe?
2. Why is `if lock.acquired: write_db()` wrong after a GC pause?
3. Safety vs liveness: which one should a consensus group keep during
 a partition?
4. Give one reason wall-clock last-write-wins corrupts a calendar app.

Continue to [Consistency and consensus](../10-consistency-and-consensus/).

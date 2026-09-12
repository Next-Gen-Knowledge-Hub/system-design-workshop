# 9. The trouble with distributed systems

Companion notes for **Chapter 9** of *Designing Data-Intensive Applications*
(2nd ed.). Chapters 6–8 described algorithms. This chapter is why those
algorithms are ugly: the network lies, clocks lie, processes freeze, and
you cannot tell which.

The same demons show up everywhere: crash, delay, pause, and
unsynchronized clocks. This chapter names them so you can design for
them on purpose — and so chapter 10’s consensus talk is not magic.

## What this chapter is for

On one computer, `x = 1` either happened or your process died. On two
computers, **partial failure** is normal: one node did the thing, the
other never heard, the packet is in a queue, or you will get the reply
in 30 seconds. Distributed system design is **tolerating partial
failure** without pretending it is rare.

If you can keep the problem on one machine, chapter 1 already told you
to consider it. This chapter is the cost of the other choice — paid for
geo latency, fault isolation, and scale.

## Faults and partial failures

A **fault** is a component misbehaving. A **failure** is when the system
as a whole stops delivering what it promised. Distributed systems try to
contain faults so they do not become service failures.

Partial failure is the defining property: some nodes work, some do not,
and the working ones cannot reliably know which is which. There is no
shared memory, no global variable, no “ask the OS for the truth about
the cluster.” The only channel is the network — and it is unreliable.

## Unreliable networks

### The limitations of TCP

TCP delivers a **reliable byte stream** *if both ends stay up and the
network eventually delivers*. It does not give you:

- A bound on **how long** a send or ack takes.
- A distinction between “peer crashed,” “peer is slow,” and “network
  dropped my packets.”
- Protection from a switch that black-holes 1% for an hour, or a
  middlebox that resets long connections.

“The request was sent” is not “the server processed it.” “No response”
is not “it did not process it.”

### Network faults in practice

Datacenter networks are usually excellent — until they are not. Real
outages include asymmetric routing, silent packet loss, GC-induced
delays that look like network stalls, cloud “network partitions”
between AZs, and the classic “I can ping you but your app port is
black-holed.” WAN links between regions are worse: higher latency,
more variance, more political and physical failure modes.

Design as if **any** message may be lost, delayed, duplicated, or
reordered (TCP prevents reorder *within* a connection; your *logical*
retries create duplicates at the application layer).

### Fault detection

Most systems do not have a perfect failure detector. They use
**heartbeats** and **timeouts**, sometimes gossip-style suspicion. A node
that answers slowly (**limping**) is harder than a node that is dead:
it is “alive” enough to confuse detectors and “dead” enough to ruin
SLOs. Declaring it dead moves load; leaving it in the rotation poisons
latency.

### Timeouts and unbounded delays

**Timeouts are guesses.** Too short: you declare a live node dead, fail
over, risk split brain, amplify load with retries. Too long: users wait.
Adaptive timeouts (percentile of recent RTT) help; they do not remove
the ambiguity.

When you time out, you are in one of three worlds:

1. The request never reached the peer.
2. The peer did the work; the response was lost.
3. The peer is still doing the work (or will).

That is why **idempotent APIs** and **dedupe keys** exist (ch. 8 / 13).
Without them, “retry on timeout” doubles charges.

### Synchronous versus asynchronous networks

A **synchronous** network model assumes bounded delay. Hard real-time
systems can buy something close — with dedicated hardware, low
utilization, and cost. The public internet and typical datacenters are
**asynchronous**: delay is unbounded. Useful algorithms assume
**partial synchrony**: eventually, long enough stretches are stable for
progress (liveness), while safety never depends on the bound being true.

If a paper’s proof needs bounded delay for **safety**, be suspicious in
cloud land.

## Unreliable clocks

### Monotonic versus time-of-day clocks

**Time-of-day (wall) clocks** answer “what is the date?” They jump when
NTP steps, when a VM pauses and resumes, when someone sets the clock.
They can go **backwards**. They differ across machines by milliseconds
to seconds (sometimes more). Do **not** use them alone to order writes
(naive last-write-wins) or to decide lock expiry without extra care.

**Monotonic clocks** (`CLOCK_MONOTONIC`, etc.) measure elapsed time
*on one machine*. Good for timeouts local to a process. They do **not**
compare across machines — there is no shared zero.

### Clock synchronization and accuracy

NTP is best-effort. Google **TrueTime** and cousins expose an
**uncertainty interval**, not a perfect timestamp. If you *must* use
wall time for ordering or commit visibility, treat uncertainty as part
of the API: **wait out the error bound** before exposing a write
(clock-bound wait). Spanner-style systems do this; your app’s
`Date.now()` does not.

### Relying on synchronized clocks

Bad uses: LWW conflict resolution across regions; assuming timestamp
order is causal order; expiring leases with only a wall-clock TTL and
no fencing. Good uses: approximate event time for analytics (with late
data handling — ch. 12); human-readable logs; legal “as of” timestamps
with stated precision.

### Process pauses

GC, swapping, “the hypervisor froze my VM,” a SIGSTOP, a long STW
pause, a container CPU throttle. The process wakes up with a **stale
view**: it still thinks it is leader, its lease TTL expired in real
time, its cache is ancient. Heartbeats did not go out, so peers
declared it dead. Then it writes anyway.

That is the **zombie** problem. The fix is not “pause less” (you will
never pause zero). The fix is **fencing** (ch. 6 / 10): a monotonic
generation / token that storage checks so a zombie’s writes are
rejected even if the zombie believes it still holds the lock.

## Knowledge, truth, and lies

There is no shared memory. The only facts you have are messages you
received. You cannot know what you were not told. Protocols must work
with **local knowledge** only.

### The majority rules

A **majority quorum** is how you make a decision that another partition
cannot also make: two majorities of `2f+1` must overlap, so they cannot
decide contradictory committed values at once. A minority that cannot
reach a quorum must **stop**, not invent a new truth. This is the honest
core behind “CP” systems during a partition — and behind Raft/Paxos
(ch. 10).

### Distributed locks and leases

`SETNX` in Redis (or a row in MySQL) is not a lock that survives pauses
by itself. You need:

- A **lease** (time-bounded; must renew) so a dead holder does not block
  forever.
- A **fencing token** (monotonic number from the lock service) that the
  **storage** checks on every write, so a zombie with an expired lease
  cannot overwrite the new holder’s work.

Without fencing, the client that paused still holds a “lock” in its own
imagination. The book’s warning is blunt: a lock that only lives in the
client’s memory is theater.

### Byzantine faults

Nodes that **lie** or return garbage — bugs, malice, bit flips that look
like valid messages. Most data systems assume **non-Byzantine**
(crash-stop or crash-recovery): you may die or reboot with disk intact;
you do not send cryptographically plausible evil responses on purpose.
Checksums and disk ECC fight corruption; crypto and blockchain spend
their lives on Byzantine agreement. Your OLTP cluster usually does not
pay Byzantine costs — except where you add verification (checksums,
signatures on the audit log).

### System model and reality

When you read a paper, it assumes a **system model**:

- What can fail (crash, omit messages, lie)?
- Can nodes recover with disk intact?
- Are delays bounded?

**Safety** vs **liveness:** safety = nothing irrevocably wrong happened
(two leaders committed different values for the same slot). Liveness =
something good *eventually* happens (a request gets a response). Under
a total partition, you often keep safety and sacrifice liveness —
refuse writes rather than guess. That is the honest reading of CAP:
during a partition you choose linearizable consistency or availability
of conflicting writes, not both.

### Formal methods and randomized testing (2nd edition)

**TLA+** / PlusCal model checking, **Jepsen**-style fault injection,
**deterministic simulation** (FoundationDB’s testing religion): ways to
find the pause/network cases humans do not storyboard. You will not
TLA+ every CRUD service. You *should* know that “we wrote a lock”
without a model or chaos test is hope, not engineering.

Deterministic simulation replaces the network, clock, and scheduler
with a controlled fake so rare interleavings become reproducible. Even
then, hash iteration order, OOMs, and other nondeterminism sneak back —
the book’s point is vigilance, not a silver bullet.

## How this shows up when you design something

- Every arrow on the whiteboard needs a timeout *and* a retry policy
  that cannot metastable-storm (ch. 2).
- Every lock needs fencing.
- Every “the replica is healthy” is a timeout, not a fact.
- Every timestamp in a conflict resolver is a confession that you
  trust clocks.
- Prefer **one machine** (or one primary) when geo and HA do not force
  distribution — complexity is a tax.

In interviews, calling out “we cannot know if the payment RPC succeeded
if we timed out” is the adult move. Idempotent APIs exist because of
that sentence.

## Check yourself

1. You sent `POST /charge` and got no response. What are the three
   worlds you might be in, and what must the API provide so retry is
   safe?
2. Why is `if lock.acquired: write_db()` wrong after a GC pause?
3. Safety vs liveness: which one should a consensus group keep during
   a partition?
4. Give one reason wall-clock last-write-wins corrupts a calendar app.
5. Why can TCP “reliability” still leave you unsure whether the server
   charged the card?
6. Monotonic clock vs wall clock: which one for “this RPC took 2 s,”
   which one for “expires at 17:00 UTC,” and what is dangerous about
   the second?
7. What is a limping node, and why is it worse than a clean crash?
8. Why does a minority partition have to refuse writes in a quorum
   system?

Continue to [Consistency and consensus](../10-consistency-and-consensus/).

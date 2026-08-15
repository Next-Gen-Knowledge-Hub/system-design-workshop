# 2. Defining nonfunctional requirements

Companion notes for **Chapter 2** of *Designing Data-Intensive Applications*
(2nd ed.). Chapter 1 asked *what kind of system*. This chapter asks *how
good does it have to be*, in words you can put on a design doc: latency,
reliability, scale, and whether humans can still operate the thing next
year.

## What this chapter is for

“Make it scalable and highly available” is not a requirement. It is a
mood. The book gives you the measuring sticks so that when you later
pick replication (ch. 6) or sharding (ch. 7), you know *which number*
you are buying.

The running example is a **social network home timeline** — the same
shape as Twitter/X, Instagram, LinkedIn feed. It is a gift for system
design practice: easy to explain, nasty at scale, and full of the
fan-out vs fan-in choice you will reuse for notifications, inboxes, and
activity feeds.

## Case study: home timelines

You have users, posts, and a follow graph. When someone opens the app
you must show a **home timeline**: recent posts from people they follow,
roughly newest first.

Two ways to build it:

**Fan-out on read (pull).** On open, query everyone they follow, merge,
sort, cut. Cheap when a user follows 50 people. Brutal when they follow
thousands, or when a celebrity posts and millions of timelines need that
post. The read path does the work.

**Fan-out on write (push).** When someone posts, you write that post into
the materialized timeline of every follower. Reads become “get this
user’s precomputed list.” Writes become expensive for celebrities
(Justin Bieber’s tweet used to occupy a noticeable slice of Twitter’s
fleet). You also have to update the list when someone follows a new
person (backfill) and when a post is deleted.

Real systems mix: push for normal users, pull for celebrities, maybe a
lossy timeline that can drop a post rather than stall (Bluesky has
written about this). The design lesson is not “always precompute.” It
is: **materializing a view moves work from read time to write time.**
That is CQRS in civilian clothes (see [ch. 3](../3-data-models/)).

You also meet **exactly-once delivery** as a reliability itch: if the
fan-out worker crashes mid-way, you must not skip followers and must not
double-insert. Chapter 12 spends real pages on that; here you only need
to notice the requirement exists.

## Describing performance

People say “latency” and mean three different things.

- **Latency** is how long a thing sits waiting (queueing, network).
- **Response time** is what the client actually feels: waiting + service
 time + client overhead. That is the number users care about.
- **Throughput** is how many requests (or bytes, or records) per second
 the system can complete.

Averages lie. A mean of 50 ms can hide a p99 of 2 s, and humans remember
the slow ones. The book wants you to talk in **percentiles**: p50
(median), p95, p99, sometimes p999. Tail latency is where queues,
GCs, and noisy neighbors live.

**SLAs / SLOs** turn those numbers into a contract: e.g. p99 read under
200 ms over a month, 99.9% successful requests. You cannot honestly
promise a *maximum* response time on the public internet; you promise a
percentile and a success rate.

When you sketch capacity, say the **load parameters** explicitly: QPS,
read/write ratio, payload size, concurrent users, fan-out factor
(“average follows = 200, celebrity follows = 20 million”). Scalability
is not a property of a system in the abstract. It is: *when this load
parameter grows, what happens to response time and cost?*

## Reliability and fault tolerance

A system is **reliable** if it keeps meeting its promises when things go
wrong. **Faults** are the things that go wrong (disk dies, process
pauses, a bad deploy). **Failures** are when the user-visible service
stops doing the right thing. **Fault tolerance** is the art of having
faults without failures.

Hardware faults are expected: disks, DIMMs, NICs, whole machines, whole
zones. At a few servers you replace boxes; at thousands, something is
always dying. Replication exists because of this (ch. 6).

Software faults are nastier because they **correlate**. One bad config,
one OOM pattern, one leap-second bug hits every node at once.
Replication of the same binary does not save you. Diversity, staged
rollouts, and “this process must not assume the world is healthy”
matter more.

Humans are in the loop: wrong query on prod, bad migration, pagination
that turns into a full table scan. Reliable organizations design for
**people making mistakes**: staging, easy rollback, least privilege,
and **blameless postmortems** so the next person is not afraid to
report the near miss.

You will also hear **metastable failures**: a small overload causes
retries, which cause more overload, which never drains even after the
original spike is gone. Backoff with jitter, load shedding, circuit
breakers, and admission control are how you stop the spiral. *Release
It!* is the usual extra reading; this chapter just wants you to know
the shape.

## Scalability

**Load** is “how much work is arriving.” **Performance** is “how it
feels / how much we complete.” **Scalability** is “can we keep
performance acceptable as load grows, by adding resources in a
predictable way?”

Three classic hardware stories:

- **Shared memory** — one box, many CPUs, one RAM. Vertical scale.
 Simple. Ceiling.
- **Shared disk** — several machines, one SAN. Some databases still
 look like this.
- **Shared nothing** — each node has its own CPU, RAM, disk; they
 coordinate over the network. This is how most large data systems
 scale (sharding, ch. 7).

Principles you will reuse: split work into independent pieces (shards,
partitions, consumers in a group), avoid single hot keys, scale *the
bottleneck* not “the architecture.” Twitter’s problem was not “we need
microservices”; it was “one user has 30 million followers.”

## Maintainability

Three faces, all boring, all what actually kills systems:

- **Operability** — can ops see health, restore from backup, rotate
 certs, understand a graph without reading the original author’s
 mind?
- **Simplicity** — accidental complexity (clever caches, hidden
 coupling) vs essential complexity (the domain really is hard).
 Good abstractions hide the first without lying about the second.
- **Evolvability** — can we change the product? Irreversible moves
 (a one-way database migration with no dual-write period) make
 evolution expensive. Chapter 5 is largely about making change
 *reversible* at the byte level (compatibility).

The rest of DDIA is a catalog of building blocks that have survived
contact with production. Use them so you are not inventing a unique
snowflake of complexity.

## How this shows up when you design something

A credible design states:

- Load: “~50k writes/s, ~500k reads/s, 1 KB payloads, 100:1 read/write.”
- Latency: “p99 200 ms for the read path; writes can be 1 s.”
- Failure: “survive one AZ; lost writes after failover: none / 1 s of
 async replica lag.”
- Evolution: “we will dual-write for two weeks.”

If those lines are missing, the boxes on the whiteboard are decoration.

## Check yourself

1. Why is p99 more honest than average response time for a user-facing API?
2. Fan-out on write vs fan-out on read: which load parameter explodes in
 each, and how would you mix them for a celebrity?
3. Give a software fault that replication *cannot* fix.
4. What would you put in an SLO for a payments authorize endpoint vs a
 “export my data” endpoint?

Continue to [Data models and query languages](../3-data-models/).

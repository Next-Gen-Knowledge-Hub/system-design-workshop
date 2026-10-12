# 2. Defining nonfunctional requirements

These notes follow Chapter 2 of *Designing Data-Intensive Applications*, 2nd edition, by Kleppmann and Riccomini.

A requirement that says "make it scalable and highly available" names a mood. This chapter turns that mood into numbers and behaviors you can write on a design: how long a user waits, how many requests finish per second, which faults the service absorbs, and whether a person can still operate and change it next year. The running example is a social-network home timeline, the same shape as the feeds on Twitter/X, Instagram, and LinkedIn. It is easy to describe and expensive at the edges, and the same push-or-pull choice shows up in notifications, inboxes, and activity feeds.

## The home timeline

There are users, posts, and a follow graph. When someone opens the app, the service builds a **home timeline**: recent posts from the people they follow, roughly newest first.

Two builds exist. They move the same work to opposite ends of the request.

### Fan-out on read

**Fan-out on read**, also called pull, does the work when the timeline is opened. The read path loads the list of people this user follows, loads recent posts from each of them, merges those lists, sorts by time, and cuts the result down to one page.

Take a user who follows 50 people. The open does about 50 small reads, a merge, and a sort. That is cheap. Take a user who follows thousands. The same open does thousands of reads before it can show a page, and the wait sits on the person holding the phone. Take a celebrity post that must appear in millions of timelines. If every one of those opens pulls that post by querying the celebrity, the celebrity's post store becomes the place every open waits.

The read path does the merge. Writes stay a single insert of the post. You pay at the moment the user is watching.

### Fan-out on write

**Fan-out on write**, also called push, does the work when the post is created. The write path looks up every follower and inserts that post into a **materialized timeline**, a list already stored per user. The open becomes "read this user's precomputed list." The merge happened earlier, when the author posted.

Walk a normal post. User 42 publishes post 900. The post row is stored. The follower list is 200 people, around the average this chapter uses as a load parameter. The fan-out worker appends post 900 to 200 timeline lists. User 17 opens the app and reads their own list. The open does not visit the other 199 followers.

A celebrity breaks that arithmetic. Justin Bieber's tweet used to occupy a noticeable slice of Twitter's fleet, because one write expanded into a timeline insert for a huge follower count. This chapter's load-parameter sketch puts a celebrity at 20 million follows. The scalability discussion of the same product puts one user at 30 million followers. Either number means one post becomes tens of millions of writes, on the path of a single publish.

Two more writes appear that pull does not have. When user 17 follows someone new, the timeline has to be **backfilled**: recent posts from that person are copied into user 17's list, or the next open is missing them. When post 900 is deleted, every timeline that received it has to drop it, or the deletion exists on the post and not on the screens that already copied it.

### Mixing them, and crashing in the middle

Production feeds mix the two. Ordinary accounts are pushed: the follower count is small, and the open should be one list read. Celebrities are pulled: followers merge the celebrity's recent posts at read time with the precomputed list, so one tweet does not enqueue tens of millions of inserts. A **lossy** timeline, the approach Bluesky has used, can drop a post rather than stall the open when the merge cannot keep up. The dropped post is the price of a page that still returns.

The general mechanism: materializing a view moves work from read time to write time. That split of write model and read model is **CQRS** (command query responsibility segregation). The command is "user 42 published post 900," stored once. The query is "user 17's home page," served from a list built for that query. The list can be rebuilt from the posts and the follow graph. It is not the authority for what was said.

**Exactly-once** delivery is the reliability requirement on the push path. Suppose the worker has followers 1, 2, and 3 to update, and the process dies after inserting post 900 into follower 1's list. A retry that starts over from follower 1 inserts a second copy for follower 1. A retry that skips ahead and starts at follower 2 never needed to skip anyone if it restarts cleanly, but a retry that guesses "1 is done" and then loses its place skips follower 2. The requirement is both halves: do not skip a follower, and do not insert the same post into the same timeline twice. Noticing that the requirement exists is the point here. Meeting it means the worker can tell, for each follower, whether that insert already landed.

What you lose on pure pull: the open is fast for 50 follows and slow for thousands, and a celebrity read concentrates millions of opens on one author's posts. What you lose on pure push: one publish expands by the follower count, a new follow is a backfill, and a delete is a second fan-out. What you lose if the worker crashes without a per-follower record of what landed: some timelines miss the post, some show it twice, and the two failures can happen in the same retry.

## Latency, response time, and throughput

People say "latency" for three different measurements.

**Latency** is time spent waiting: sitting in a queue, or waiting on the network. **Response time** is what the client actually measures. It adds three pieces: that wait, the time the service spends doing the work, and overhead on the client itself. Users remember response time. **Throughput** is how many requests, bytes, or records the system completes per second. A path can have a short response time at a trickle and a long one when the queue grows, so a throughput number without a response-time number does not say what the open feels like.

Averages hide the waits people remember. A mean response time of 50 ms can sit next to a p99 of 2 seconds. Sort the requests. The **p50** (the median) is the time that half the requests beat. The **p95** is the time that 95 percent beat. The **p99** is the time that 99 percent beat. The **p999** is the 99.9th percentile. **Tail latency** is that far end. It is where requests sit in queues, where a garbage-collection pause freezes a process, and where a **noisy neighbor** — another tenant on the same machine — steals CPU or disk.

An **SLO** (service level objective) is the target you operate to. An **SLA** (service level agreement) is that kind of target turned into a promise to someone else. A usable form is: p99 of timeline reads under 200 ms over a month, and 99.9 percent of requests succeed. A maximum response time on the public internet is not an honest promise. A packet can be delayed by networks you do not control. You promise a percentile and a success rate.

What you lose if you publish the mean: the design looks fine at 50 ms while one request in a hundred takes 2 seconds, and that is the request the user retells. What you lose if the contract says "never slower than 200 ms": the first congested path on the public internet breaks the contract, even though the service did the work.

## Load, and what scalability means

**Load** is how much work is arriving. **Performance** is how that work feels and how much of it finishes. **Scalability** is whether you can keep performance acceptable as the load grows, by adding resources in a way you can predict. It is not a property a system has in the abstract. It is a sentence of the form: when this load parameter grows, what happens to response time and to cost?

Name the **load parameters** in numbers. Queries per second (QPS). The read/write ratio. The payload size. How many users are active at once. The **fan-out factor**: how many followers one write touches. A timeline sketch from this chapter: average follows = 200, celebrity follows = 20 million. A credible capacity line looks like: about 50,000 writes per second, about 500,000 reads per second, 1 KB payloads, a 100:1 read/write ratio. The matching latency line: p99 of 200 ms on the read path, and writes allowed to take 1 second. The matching failure line: survive the loss of one availability zone (an **AZ**, one isolated site inside a region); after failover, lost writes are either none, or bounded by 1 second of asynchronous replica lag. The matching change line: dual-write the old and new shapes for two weeks.

**Replication** is why the failure line mentions a replica. A replica is another node applying the same writes. **Asynchronous** replication means the primary acknowledges the write before the replica has it, so the replica can be behind. That gap is **replica lag**. **Failover** promotes a replica when the primary is gone. Writes that reached the primary and had not reached the replica are the lost writes. "None" means you waited for the copy before acknowledging. "1 second of async replica lag" means you accept that window.

What you lose if the parameters are missing: "scalable" has nothing to grow. The parameter that actually explodes on a timeline is the fan-out factor, from 200 to tens of millions, and a design that adds application servers does not shrink that one user's follower list.

## Faults that stay inside the promise

A system is **reliable** when it keeps meeting its promises while things go wrong. A **fault** is the thing that went wrong: a disk dies, a process pauses, a bad deploy goes out. A **failure** is the user-visible service stopping doing the right thing. **Fault tolerance** is arranging faults so they do not become failures.

Hardware faults are expected. Disks, memory modules (DIMMs), network cards (NICs), whole machines, and whole zones die. At a handful of servers you replace the box. At thousands of servers something is always dying, which is why a second copy of the data exists on another machine.

Software faults are worse because they **correlate**. One bad configuration, one out-of-memory pattern, one leap-second bug in time handling, hits every node that runs that binary, at the same moment. A replica of the same binary has the same bug. The copy does not save you. What helps is diversity of versions or of failure domains, a staged rollout so one slice sees the bug first, and processes that keep working when a dependency is sick instead of assuming the surrounding world is healthy.

People are in the path too. Someone runs the wrong query on production, ships a bad migration, or ships a pagination bug that turns a "next page" into a full table scan. A reliable organization plans for those mistakes: a staging environment that matches production closely enough to catch the scan, a rollback that is easy to perform, least privilege so the wrong query cannot touch every table, and **blameless postmortems** so the next person reports the near miss instead of hiding it.

A **metastable failure** is an overload that keeps itself alive. A small spike fills a queue. Clients retry. The retries are more load than the spike was. The queue never drains, even after the original spike is gone. Four mechanisms break the spiral. **Backoff with jitter** waits longer before each retry, plus a random extra delay, so the clients do not retry in lockstep. **Load shedding** drops work the service cannot do, so the rest still finishes. A **circuit breaker** stops calling a dependency that is already failing, instead of spending every request on a timeout. **Admission control** refuses new work at the door, before the queue becomes the outage.

What you lose: hardware replication of one binary does nothing for a bug in that binary. A retry policy with no backoff turns a short spike into a standing overload. A postmortem culture that hunts a culprit hides the next pagination bug until it is a scan on the primary.

## Where the extra resources go

Three hardware arrangements show up when you add capacity.

**Shared memory**: one machine, many CPUs, one pool of RAM. This is vertical scale. The programming model stays simple, because every CPU sees the same memory. There is a ceiling: one motherboard only gets so large.

**Shared disk**: several machines compute, and they share one storage array (a SAN, a storage area network). Some databases still look like this. The disk array is both the way they share bytes and a single place the whole group can stall.

**Shared nothing**: each node has its own CPU, its own RAM, and its own disk, and the nodes coordinate over the network. This is how most large data systems grow. The usual split is **sharding**: each node owns a subset of the keys, and a request goes to the node that owns that key. The same idea appears as partitions of a log, and as consumers in a group: several readers share the stream so each record is handled by one of them, and you add readers to add throughput.

The reusable rules are small. Split work into pieces that can proceed independently. Avoid a single hot key, a key that every request must touch. Add capacity at the bottleneck you measured, rather than adding a new architectural style and hoping the bottleneck moves. Twitter's crunch was a hot key in disguise: one user had 30 million followers. More microservices did not shrink that follower list. The fan-out of that one account did.

What you lose on shared memory: a hard ceiling, and one machine's death is the whole service. What you lose on shared disk: the array couples the machines that were supposed to be independent. What you lose on shared nothing: the network is now part of every coordination, and a hot key still lands on one node no matter how many nodes you buy.

## Keeping the system changeable

Three properties decide whether the system is still operable next year. They are ordinary, and they are what actually ends systems.

**Operability** is whether the on-call engineer can see health, restore from a backup, rotate a certificate, and understand a dashboard without reconstructing the original author's intent.

**Simplicity** separates accidental complexity from essential complexity. Essential complexity is the domain: a follow graph with celebrities really is uneven. Accidental complexity is the extra machinery: a cache whose invalidation nobody can state, a hidden coupling between two services. A good abstraction covers the accidental part and still tells the truth about the essential part.

**Evolvability** is whether the product can change. An irreversible step makes the next change expensive. A one-way database migration with no **dual-write** period is the example: dual-write means that for a while (the design line above says two weeks) every update is written in the old shape and the new shape, so a rollback still has a complete old copy. Skip that period, and the only way back is a restore. At the level of bytes, evolution stays possible when a newer reader can still read older records and an older reader can tolerate newer records. That is a property of the encoding, and it is the difference between a rolling deploy and a moment when every process must switch at once.

The building blocks that have survived production are worth using, because a design invented from scratch tends to accumulate accidental complexity that those blocks already paid for.

What you lose: a system nobody can restore is one backup drill away from a permanent outage. A clever cache with an unstated invalidation rule fails as a wrong timeline, which is worse than a slow one. A migration that cannot roll back turns a bad deploy into a data repair.

## Recap

A home timeline is recent posts from the follow graph, newest first. Fan-out on read merges at open time: cheap at 50 follows, brutal at thousands, and a celebrity post becomes a read storm. Fan-out on write inserts the post into each follower's materialized list: the open is one read, the publish costs the follower count, a new follow is a backfill, and a delete must fan out again. Mixing them pushes ordinary users and pulls celebrities. A lossy timeline can drop a post to avoid stalling. The push worker has an exactly-once problem: a crash in the middle must neither skip a follower nor insert the post twice. Materializing the list is CQRS: the command is the post, the query is a view built for the open.

Latency is waiting. Response time is waiting plus service time plus client overhead, which is what people feel. Throughput is completions per second. A mean of 50 ms can hide a p99 of 2 seconds, so the numbers to publish are percentiles: p50, p95, p99, p999. An SLO or SLA promises a percentile and a success rate, for example p99 under 200 ms over a month and 99.9 percent success, because a hard maximum on the public internet is not a promise you can keep.

Scalability is what happens to response time and cost when a named load parameter grows: QPS, read/write ratio, payload size, concurrency, fan-out. A timeline's dangerous parameter is follows, from an average of 200 to a celebrity at 20 million, and in the extreme one account at 30 million. A design states the load (about 50,000 writes/s, 500,000 reads/s, 1 KB, 100:1), the latency (p99 200 ms reads, writes up to 1 s), the failure budget (one AZ, lost writes none or 1 s of async replica lag), and the evolution plan (dual-write for two weeks).

A fault is the breakage; a failure is the user-visible miss; fault tolerance is faults without failures. Hardware dies constantly at large scale, so replicas exist. Software faults hit every copy of the same binary at once, so replication of that binary does not help. People ship bad queries, bad migrations, and pagination that becomes a table scan; staging, rollback, least privilege, and blameless postmortems are how that stays a fault. Metastable overload is retries feeding the queue that caused the retries; backoff with jitter, load shedding, circuit breakers, and admission control are how the queue drains.

Shared memory scales up inside one RAM space until the box ends. Shared disk shares a SAN. Shared nothing gives each node its own disk and scales by sharding, partitions, or a consumer group, and it still loses if one hot key owns the work. Operability, simplicity, and evolvability are whether next year's operator can see, understand, and change the system, including rolling back a migration that dual-wrote.

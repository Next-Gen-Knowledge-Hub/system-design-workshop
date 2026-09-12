# 1. Trade-offs in data systems architecture

Companion notes for **Chapter 1** of *Designing Data-Intensive Applications*
(2nd ed.). The book’s opening move is not “here is the best database.” It is:
almost every architecture question has several honest answers, and your job
is to name the trade-off before you pick.

Read the chapter, then use this walkthrough to keep the vocabulary straight.

## What this chapter is for

System design interviews and real designs both fail the same way: someone
jumps to Kafka, or Kubernetes, or “we’ll shard Postgres,” without saying
*what kind of system this is*. Chapter 1 gives you four forks that show up
in almost every design:

1. Operational vs analytical
2. Cloud vs self-hosting
3. Distributed vs single-node
4. Business goals vs law and the people in the data

If you can place a problem on those four axes, the rest of the book (and
the rest of this workshop) has somewhere to hang.

## Operational versus analytical systems

Two audiences share one company’s data, and they want opposite things.

**Operational systems (OLTP)** are the product. A user taps “pay,” a rider
requests a trip, a post is liked. The typical query looks up a *small*
number of records by key, maybe updates them, and must answer in
milliseconds. The application both reads and writes. This is what most
backend engineers mean by “the database.”

**Analytical systems (OLAP)** are for people who look at the data rather
than generate it: business analysts, data scientists, ML training jobs.
A typical query *scans* huge numbers of rows and computes counts, sums,
averages (“revenue per city in January”). They almost never mutate the
source records; they build *derived* datasets on top.

Keeping these on the same database is tempting and usually a bad idea.
A warehouse scan that touches last month’s orders will fight with the
checkout path for disk and CPU. The storage layout that is fast for
point lookups (see [ch. 4](../4-storage-and-retrieval/)) is the wrong
layout for “sum this column over a billion rows.” So organizations copy
data out of OLTP into a **data warehouse** (structured, modeled for
SQL analytics) or a **data lake** (cheaper, more flexible files on object
storage — Parquet, Avro, images, embeddings, whatever). The copy is filled
by **ETL** (extract–transform–load) or its modern cousins (ELT, reverse ETL).

Two roles sit on the bridge: **data engineers** keep the pipes and
platforms honest; **analytics engineers** model the data so analysts can
query it. You do not have to become either, but a system design that
ignores “how does this data get used for reporting / ML” is incomplete.

**Systems of record vs derived data** is the other half of this split.
A *system of record* is the authoritative place a fact is written (the
order table). A *derived* dataset is a copy transformed for another
purpose (search index, cache, warehouse table, ML feature store). Most
large products are a system of record plus several derived systems. That
idea returns as the backbone of [ch. 11–13](../11-batch-processing/).

```
Users ──► OLTP (system of record)
 │
 │ ETL / CDC / events
 ▼
 Warehouse / lake / search / cache (derived)
 │
 ▼
 Analysts, ML, other services
```

## Cloud versus self-hosting

“Should we use the managed thing?” is a business question wearing a
tech costume.

The spectrum the book draws:

- You write it and run it (maximum control, maximum toil).
- You run off-the-shelf software yourself — on your metal or on cloud VMs
 (IaaS). Self-hosting Postgres on a VM is still *you* operating Postgres.
- You call a vendor API / SaaS and they operate the software.

Cloud is not automatically cheaper. If load is **steady** and you already
know how to run the system, owning machines is often cheaper. Cloud wins
when load **spikes**, when you do not want to staff a specialty (Kafka
on-call, warehouse ops), or when you need a capability next week rather
than after a hiring cycle. You still need an operations team in the
cloud — the work moves from “rack a box” to “IAM, cost, failure modes of
someone else’s control plane.”

**Cloud-native architecture** in this book mostly means: **disaggregate
storage and compute**. Classic databases kept data on the same box that
ran queries. Aurora-style and Snowflake-style systems put durable bytes
on replicated object / block storage and let compute nodes come and go.
That is why they can scale storage and query power independently, and
why “the disk on this VM died” is a different failure than it used to be.

Kubernetes is *how you place
processes*. It does not replace the data-system trade-offs in this
chapter.

## Distributed versus single-node

A **distributed system** is several machines talking over a network.
Each participant is a **node**. You might distribute because:

- The problem is already distributed (two users on two phones).
- Cloud services and microservices *are* a network (data lives in
 service A, processing in service B).
- You want **fault tolerance** (one machine dying is not the product
 dying) or **geo latency** (data near the user).
- One machine is not big enough.

The book’s warning, which this workshop will repeat: **do not distribute
for sport**. One well-run server (or one primary with replicas you
understand) is simpler, more deterministic, and easier to debug than a
cluster. Partial failure — some nodes up, some down, network lying —
is the tax you pay the moment you add a second machine. Chapter 9 exists
because of this paragraph.

**Microservices and serverless** push you into the distributed column
even when the data would have fit on one box. Each service with its own
database makes *cross-service* consistency *your* problem (see
[ch. 8, distributed transactions](../8-transactions/)). Sometimes that
is the right team boundary. Sometimes it is an accidental distributed
monolith.

## Data systems, law, and society

This is not a “soft” appendix. GDPR (and cousins like CCPA, plus AI
regulation such as the EU AI Act) change what you are allowed to store,
for how long, and for which purpose. The **right to erasure** collides
with append-only logs and with ML models trained on personal data —
two techniques this book otherwise loves.

**Data minimization** (*Datensparsamkeit*): collect for a stated purpose,
do not keep it “in case it is useful later.” That fights the “big data”
instinct. Once you add the cost of a breach, a compelled handover, or a
fine, some data is simply not worth storing.

PCI, SOC 2, and similar audits are how buyers force this into vendor
selection. Chapter 14 picks the ethics thread back up; here the point is
architectural: immutability, derived datasets, and “keep everything”
are *design* choices with legal consequences.

## How this shows up when you design something

Before you draw boxes, say out loud:

- Is this path **OLTP** (user waiting) or **OLAP** (scan / train / report)?
- Is the store a **system of record** or a **derived** view?
- Are we paying for **elasticity**, or do we have steady load and ops skill?
- Do we *need* more than one node, and for which of: size, fault
 tolerance, geo?
- What personal data is in the request, and what is the deletion story?

If you cannot answer those, you are not ready to choose Kafka vs Postgres.

## Check yourself

1. Give one example from a product you know of an OLTP query and an OLAP
 query on *related* data. Why should they not share one box?
2. When is self-hosting the cheaper, more honest choice?
3. Name two reasons to distribute that are *not* “the data does not fit.”
4. Why is “append-only log” in tension with “user asked us to delete
 their data”?

Continue to [Defining nonfunctional requirements](../2-nonfunctional-requirements/).

# 1. Trade-offs in data systems architecture

These notes follow Chapter 1 of *Designing Data-Intensive Applications*, 2nd edition, by Kleppmann and Riccomini.

A data system stores facts and answers questions about them. Almost every architecture question has more than one honest answer, and each answer spends something different: disk layout, staff time, failure behavior, or a legal limit on what you may keep. This chapter is the four choices you name before you pick a product: who the query is for, who runs the machines, how many machines must talk, and what you are allowed to store.

## Operational systems and analytical systems

An **operational** system, also called OLTP (online transaction processing), is the database the product uses while a person is waiting. A user taps pay, a rider requests a trip, or a post is liked. The application looks up a small number of records by key, maybe updates those records, and has to answer in milliseconds. The same application both reads and writes. When a backend engineer says "the database," this is the system they usually mean.

An **analytical** system, also called OLAP (online analytical processing), serves people and jobs that look at data they did not just write: a business analyst, a data scientist, a model-training job. A typical query scans a huge number of rows and computes a count, a sum, or an average, such as revenue per city in January. The query almost never changes the source records. It reads a **derived** dataset, a copy that has been reshaped for that question.

The two workloads want different layouts on disk, and they fight when they share one. Checkout of one order is a point lookup: find one key, read that record, return. A scan of last month's orders reads a large stretch of disk and CPU to sum a column. The page layout that makes the point lookup a short seek is a poor layout for summing one column over a billion rows, because the scan wants to read that column straight through, and the lookup wants a short path to one whole row. While the scan holds the disk, the checkout waits.

So the operational store stays small and fast, and a copy is made for analysis. A **data warehouse** is a structured store whose tables are modeled for SQL aggregates. A **data lake** is a pile of cheaper, more flexible files on object storage: Parquet tables, Avro records, images, embeddings, and other files that do not share one schema. Three jobs fill the copy.

- **ETL** (extract, transform, load) reads the operational data, reshapes it, and writes the result into the warehouse or the lake.
- **ELT** loads the data first and runs the transform inside the destination.
- **Reverse ETL** copies a modeled result back into an operational tool, such as a support screen that shows a score computed in the warehouse.

**Data engineers** keep those pipes and the platforms under them working. **Analytics engineers** turn the copied tables into models an analyst can query without learning the operational schema. A design that never says how reporting or training will read the data is missing half the system.

### The system of record and the copies

A **system of record** is the one place a fact is written as the authority. If two copies disagree, this one wins. The order table is a system of record. A derived dataset is a copy transformed for another reader: a search index, a cache, a warehouse table, a feature store that holds inputs for a model. A large product is usually one system of record plus several of those copies.

Walk one order through that path. Checkout inserts a row: order 1842, user 17, city Tehran, amount 40. That insert is the system of record. Something then carries the change outward. The carrier might be a scheduled ETL job, an event the checkout service emits on purpose, or **CDC** (change data capture). CDC tails the database's own change log and republishes each insert, update, and delete, so a downstream copy can apply the same change without scraping tables. The warehouse receives a row it can sum. Search receives a document it can query by text. A cache receives the hot order so the next page does not hit the primary. Analysts, training jobs, and other services read those copies. They do not write the order row.

```
Users --> OLTP system of record (order 1842 is written here)
            |
            | ETL, ELT, CDC, or an event
            v
          warehouse / lake / search / cache   (derived copies)
            |
            v
          analysts, training jobs, other services
```

The checkout path wrote one row and returned in milliseconds. The January-revenue question sums the warehouse copy. The search box reads the search copy.

What you lose if you skip the split: the revenue scan and the checkout compete for the same disk and CPU, and checkout latency moves with whatever an analyst happened to run. What you lose if you treat a copy as the authority: a stalled pipeline serves a stale order, and there is no row left that can settle the argument. What you lose if you never make the copy: "revenue per city" becomes a billion-row scan against the product database.

## Who runs the software

The next choice is who operates the system. It is a cost and staffing question that shows up wearing a product name. There are three stops on the spectrum.

You write the software and you run it. You can change any behavior. You also own every backup, every page, and every night call.

You run off-the-shelf software yourself, on machines you own or on virtual machines you rent. Renting the machine is **IaaS** (infrastructure as a service): you get a computer, and the software on it is yours to operate. Postgres on a cloud virtual machine is still your Postgres. You patch it, you take the backups, and you explain a stuck query.

You call a vendor's API. That is **SaaS** (software as a service). The vendor runs the software. You send requests and pay the bill.

A steady load, on a system the team already knows how to run, is often cheaper on machines you own. The cloud bill wins in three situations: the load spikes, and you want machines for the spike and not for the quiet month; you do not want to staff a specialty such as Kafka on-call or warehouse operations; or you need the capability next week rather than after a hiring cycle. An operations team still exists after you move. The work shifts from racking a box to identity and access rules (IAM), cost, and the ways someone else's control plane can fail while your data is fine.

**Cloud-native**, as this chapter uses the term, means storage and compute are **disaggregated**: the durable bytes and the processes that query them live on different machines. A classic database kept the files on the same box that ran the queries. When that disk died, the queries died with it, unless a replica you understood had the bytes. Aurora-style and Snowflake-style systems put the durable bytes on replicated object storage or block storage, and they let compute nodes appear and disappear. Storage can grow without adding query CPUs. Query power can grow without copying the dataset onto each new node. A disk dying on one virtual machine is then a compute failure: another node can read the same stored bytes. That is a different failure from "the only copy lived on the disk that just died."

Kubernetes places processes onto machines and restarts them when they exit. The data choices stay yours: whether the path is operational or analytical, whether the bytes live with the query or apart from it, and what a half-reachable node means.

What you lose: a steady workload on a usage-priced service can cost more than machines you already operate well. Owned machines sized for a spike sit idle the rest of the month, and owned machines sized for the average fall over at the spike. A managed control plane can refuse calls while the bytes are healthy, and you cannot rack a replacement for an API.

## One machine and several machines

A **distributed system** is several computers cooperating by messages over a network. Each computer in that set is a **node**.

Distribute when you can name the reason.

The problem is already on more than one computer. Two users on two phones are already two machines. Any system between them has to accept that a message takes time and sometimes does not arrive.

The design is already a network. A cloud service, and a split into microservices, is a network even when the rows would fit on one disk. Service A holds the data. Service B does the work. Every user action pays a round trip, and the call can fail while the other process is healthy.

You want **fault tolerance**: one machine dying should leave the product up. Or you want the bytes near the user, so a read does not cross an ocean. The mechanism under both is **replication**: another node keeps a copy of the same writes, so a read or a failover has somewhere to go. A **primary** accepts the writes. A **replica** applies the same writes. If you have tested what the replica knows and how far behind it is, one primary plus replicas is still a much smaller story than a cluster of peers.

Or one machine is not big enough. The data does not fit on its disk, or the query rate does not fit on its CPUs.

A single well-run server is simpler to debug than a cluster. Inside one process, a call returns or the process is dead. The moment a second machine is added, you pay **partial failure**. Some nodes are up, some are down, and the network can lie: a dropped packet looks, for a while, like a node that is slow. Waiting a fixed time cannot separate "slow" from "gone." A retry can apply the same write twice when the first attempt actually landed and only the response was lost.

**Microservices** and **serverless** functions put the design in that distributed column even when the data would have fit on one box. Each service with its own database splits a fact that used to be one local write. **Cross-service consistency** is then yours to define: the payment write and the order write are two writes on two machines, and you need a story for the case where one landed and the other did not. A **distributed transaction** tries to make those writes commit or abort together. The coordinator and the network sit on the path of the write, and an unreachable coordinator is a new way for a simple purchase to stick. Sometimes the split matches a team boundary, because two teams need to ship without sharing one database. Sometimes it is a **distributed monolith**: no service can finish a user request alone, every request still needs all of them, and you have added partial failure without a team boundary that needed it.

What you lose if you add a second node with no size, fault, or geography reason: partial failure, and no capacity you were short of. What you lose if two services write two databases with no consistency story: the payment exists and the order does not, or the reverse, and the user sees one of those.

## What the law does to a stored fact

GDPR, similar laws such as the CCPA, and AI regulation such as the EU AI Act change what you may store, how long you may keep it, and which purpose you may use it for. The **right to erasure** says a person can require their personal data to be deleted. That requirement hits two designs that are otherwise useful.

An **append-only log** writes new records at the end and does not edit old bytes in place. A `DELETE` of the operational row does not reach a person who is still sitting in an older log record. Removing them means rewriting the log, and it means finding every other copy. A model trained on personal data has the same shape: deleting the row leaves the person in the training set the model was built from. Erasure has to reach every copy, not only the row a `DELETE` can see.

**Data minimization** (*Datensparsamkeit*) means you collect a field for a stated purpose, and you do not keep it because it might be useful later. That rule fights the habit of storing every event on the grounds that disk is cheap. Add the cost of a breach, a compelled handover, or a fine, and some columns cost more to hold than they are worth.

PCI, SOC 2, and audits like them are how a buyer forces the same question into a vendor choice. The buyer asks where a card number lives, who can read it, and how long a backup keeps it. Immutability, derived datasets, and a policy of keeping everything are design choices with legal weight: every extra copy is another place a deletion has to arrive, and an append-only log is a copy that refuses an ordinary update.

What you lose: a deletion that touches only the order table leaves the person in the warehouse, the search index, the feature store, and last quarter's backup. A model trained last month still answers from data the person withdrew. A log you cannot edit becomes the incident the first time erasure is enforced.

## Recap

An operational path looks up or updates a few records by key while a user waits, and it answers in milliseconds. An analytical path scans many rows and reduces them to a count, a sum, or an average. The scan belongs on a copy, because it wants a column laid out for a sequential read, and because it would take the disk and CPU the checkout path needs. The order row is the system of record. The warehouse, the lake, search, the cache, and the feature store are derived. ETL, ELT, reverse ETL, CDC, and emitted events are the ways the change gets from the row to the copy.

Writing and running the software, or running Postgres yourself on a rented virtual machine, leaves operations with you. A vendor API moves the daily work onto identity rules, cost, and that vendor's control plane. Steady load you already know how to run is often cheaper on machines you own. Spikes, a missing specialty, and a need measured in days are where the cloud price buys time. Disaggregated storage and compute let the bytes and the query CPUs grow on separate schedules, and a dead disk on a compute node is no longer the loss of the only copy. Kubernetes places processes; it does not make those data choices.

A second machine is justified when the users are already on separate devices, when the services are already a network, when one death must not be an outage, when the bytes must sit near the user, or when one machine cannot hold the data or the rate. Partial failure arrives with that second machine: some nodes up, some down, the network able to drop a reply. Two databases under two services make cross-service consistency, and any distributed transaction around it, part of the write. GDPR, the CCPA, and the EU AI Act make every append-only log, every derived copy, and every trained model part of the deletion path, and data minimization says some fields should never be written down.

# 7. Sharding

Companion notes for **Chapter 7** of *Designing Data-Intensive Applications*
(2nd ed.). The first edition called this **partitioning**. Same idea:
when one replica set cannot hold the data or the **write** QPS, you
**split** the dataset across independent groups of nodes. Each piece
is a **shard** (partition). Replication (ch. 6) copies the same data;
sharding splits different data.

Normally each record belongs to **exactly one** shard. That shard is
usually **replicated**, so the record still lives on several machines
for fault tolerance. A node may store many shards: leader for some,
follower for others. The picture is a chessboard, not three giant
replicas of the whole database.

## What this chapter is for

Sharding buys **scale** (data size and write throughput) and sometimes
**tenancy isolation**. It costs: cross-shard queries, cross-shard
transactions (ch. 8), rebalancing, and a routing layer. Hot spots can
make ten nodes behave like one.

Read scale alone does **not** require sharding — replicas (ch. 6) are
the first lever. Shard when **one leader cannot take the writes** or
**one node cannot hold the bytes**.

The choice of sharding scheme is mostly independent of the replication
style. This chapter ignores replication on purpose; in a real design
you still draw both.

## Names, and what this is not

Vendors do not share a word:

| You might hear | In |
|---|---|
| Partition | Kafka |
| Range | CockroachDB |
| Region | HBase, TiDB |
| vBucket | Couchbase |
| vnode / token-range | Riak / Cassandra |
| Tablet | Bigtable, Yugabyte, Scylla |

**PostgreSQL partitioning** is usually *files on one machine* (drop a
partition = delete a file fast). **Sharding** is *across machines*.
Some products use both words for the same thing. This folder means
**across machines**.

Unrelated: a **network partition** (ch. 9) is a broken link, not a
piece of the keyspace.

The word *shard* itself is folklore (Ultima Online’s split worlds, or
an old acronym). You only need the idea: **one of several parallel
slices of the data**.

## Combining sharding and replication

```
          shard 1                  shard 2                  shard 3
       ┌────────────┐           ┌────────────┐           ┌────────────┐
node A │ LEADER     │           │ follower   │           │ follower   │
       └────────────┘           └────────────┘           └────────────┘
       ┌────────────┐           ┌────────────┐           ┌────────────┐
node B │ follower   │           │ LEADER     │           │ follower   │
       └────────────┘           └────────────┘           └────────────┘
       ┌────────────┐           ┌────────────┐           ┌────────────┐
node C │ follower   │           │ follower   │           │ LEADER     │
       └────────────┘           └────────────┘           └────────────┘
```

Each shard is a mini-database with its own leader. Writes for a key
go to **that** shard’s leader. You have scaled writes **if** keys
spread. You have not scaled a key that everyone hits.

## Pros, cons, and why teams shard too early

**Pros**

- Horizontal scale: add boxes, add capacity, if load is even.
- Smaller blast radius: one shard’s compaction storm or restore is
  not the whole dataset.
- Possible **geo** placement of a tenant or a key range.
- Operations that are cheap per shard (drop old data, compact, vacuum)
  stay bounded.

**Cons**

- You cannot `JOIN` or `UNIQUE` across the whole dataset without extra
  machinery (scatter-gather, global indexes, or a txn manager — ch. 8).
- Schema change, backup, rolling restart, and “run this backfill”
  multiply by shard count.
- Picking a **shard key** is a one-way-ish decision. Changing it means
  rewriting the cluster.
- Cross-shard writes: one shard commits, another fails — partial
  success. That is the next two chapters, not a footnote.

**Reads vs writes.** If the pain is reads, try replicas, caches, and
better indexes (ch. 4) first. Sharding is how you split **leaders**.

## Sharding for multitenancy

SaaS products often shard by `tenant_id` so one customer’s data and
load sit in a box you can enlarge, migrate, or isolate.

What that buys:

- **Noisy-neighbor control** — a heavy tenant does not eat everyone’s
  buffer pool. Cells / pods of tenants are the same idea at a coarser
  grain (AWS “cell-based” architecture).
- **Compliance** — deleting one tenant, or pinning them to an EU
  region, is a shard operation instead of a table scan of the world.
- **Schema rollout** — you can migrate tenant-by-tenant instead of
  one giant `ALTER`. (You still need a story for *in-flight* tenants;
  products exist specifically for transactional schema change across
  tenant databases.)

Ugly cases:

- One tenant is 90% of traffic — they *are* a hot spot. They need
  their own shard (or several), not “more shared shards.”
- Tiny tenants: thousands of almost-empty shards waste connections
  and file handles. Pack many small tenants per shard; split when
  they grow.
- Queries that ignore tenant (“all unpaid invoices in the country”)
  become scatter-gather or a warehouse job. That is often the right
  answer, not a reason to unshard OLTP.

Citus-style **schema-based sharding** (one schema per tenant) is this
pattern with Postgres vocabulary.

## Sharding of key-value data

You need a function: `key → shard`, cheap to compute, stable until
you *intentionally* move data.

Often the key is **compound**: a **partition key** (which shard) plus
a **sort key** (order *inside* the shard). Then “all events for this
user, time range” is a local range scan. “All events in this hour,
every user” is not.

### By key range

Keys are sorted; shard 1 owns `A–F`, shard 2 `G–M`, … Range scans
(“all orders whose ids fall in today,” if ids are time-sortable)
stay on few shards. Adjacent keys share a node — good for locality,
bad when adjacent keys are **all hot**.

**Hot ranges** are the failure mode: sequential ids, latest
timestamp, a popular prefix (`status=new`). The newest range takes
all the inserts; nine other shards sit idle. This is why “UUID
primary key + range sharding” and “timestamp as the partition key”
need a second look.

Systems that **auto-split** a range that grew too big or too hot
(HBase, Cockroach, Mongo range, Bigtable tablets) exist because
humans guess boundaries badly. Split is not free: it rewrites files
(think LSM compaction) and often hits the shard that is **already**
overloaded. Plan for split cost, or you split during the incident.

### By hash of key

Hash the partition key, then map the hash onto shards. A good hash
turns skewed *keys* into a flat space: similar strings do not land
near each other. Same input always hashes the same — this is not
cryptography. Mongo historically used MD5; Cassandra/Scylla use
Murmur3. **Language `hashCode()` is not safe** across processes,
versions, or languages; do not shard on it.

You buy even load for point gets. You pay: **range scans become
scatter-gather** unless the range is *inside* one partition key
(sort key). “All orders today” with `hash(order_id)` hits every
shard.

**`hash(key) % node_count` is a disaster when you add a node.**
Almost every key’s remainder changes, so almost all data moves, and
systems that order by partition (Kafka) also **break ordering**.

Usual fixes — all of them **decouple “which shard” from “which
node”**:

1. **Fixed shard count.** Hash to 1024 (or 4096, …) virtual shards;
   map shards → nodes. Add a node by **moving whole shards**, not
   remapping keys. Kafka’s topic partitions are this: changing
   partition count **reshuffles keys**. Pick a number you can live
   with.
2. **Consistent hashing** (the ring, often with **virtual nodes** per
   physical box). Adding a node steals a slice of the ring from
   neighbors instead of reshuffling everything. Cassandra/Scylla
   look like this (many token ranges per node so luck averages out).
3. Other algorithms with the same two promises — **balanced load**,
   **minimal key movement when N changes** — e.g. **rendezvous
   (highest random weight)** and **jump consistent hashing**. Some
   move *individual keys* onto the new node rather than splitting
   ranges. Which is better depends on whether you want to move a
   few big files or many small key ranges.

“Consistent” here is **not** ACID consistency. It means a key tends
to **stay on the same shard** when the cluster grows.

### Skewed workloads and relieving hot spots

Hashing spreads *keys*. It does not spread a **single key**. A
celebrity `user_id`, a flash-sale `product_id`, a `status = 'new'`
queue everyone polls: one partition, one leader, the rest of the
cluster applauds.

Tricks, all imperfect:

- **Isolate** the hot key on its own shard / dedicated hardware
  (range-based systems can put one key in a shard by itself).
- **Cache** the hot read path so the shard is not the website.
- **Split writes** with a random suffix (`product_id + 00..99`).
  Writes fan out; **reads must gather and merge**. Track which keys
  are in this special mode; viral load is temporary, so you want
  a way to **unsalt** later.
- **Change the product** so the hot path is not one row (fan-out
  on write for the celebrity — ch. 2).

Cloud systems market **adaptive capacity / heat management**. Treat
that as “they will try to move load,” not as a substitute for a
sane key.

Write-hot and read-hot keys need different treatments. A viral post
is often read-hot (cache). A counter everyone increments is
write-hot (shard the counter or accept a single-leader bottleneck).

### Rebalancing: automatic versus manual

When you add or remove nodes, **some data must move**. Two schools:

**Move whole shards** between nodes (fixed shard count). The
key→shard map stays put; only shard→node changes. Simple routing
cache. You need enough shards that you can give a new node a fair
share without splitting mid-flight.

**Split and merge ranges** (key-range or hash-range). More elastic;
split cost as above.

**Automatic rebalance** is convenient and can autoscale. It is also
how you get a **cascading failure**: one node is slow → declared
dead → cluster moves its shards onto everyone else → they slow
down → more false deaths. Rebalance is heavy (network + disk) and
must continue to **take writes**. If you are already at write
ceiling, the split cannot keep up.

**Manual / planned rebalance** is slower and calmer. Use it before
known spikes (Cyber Monday, ticket drops). A human in the loop is
not anti-automation; it is a circuit breaker on a dangerous robot.

Either way: do not move **random keys** one by one if you can move
**whole shards**. The map stays explainable.

## Request routing

The client must hit a node that **owns that key’s shard**. A Kubernetes
Service or a generic load balancer sends traffic to *any healthy
pod*. That is correct for **stateless** app servers and **wrong**
for a sharded store unless every pod is a router.

Three patterns (you will draw one of them):

1. **Any node, then forward.** Client hits a random replica; if it
   is not the owner, it proxies. Gossip or a local copy of the map
   must be good enough. Extra hop.
2. **Routing tier** (`mongos`, Vitess vtgate, many proxies). Clients
   are dumb; the proxy is shard-aware. The proxy fleet becomes a
   thing you shard and debug.
3. **Smart client.** The client has the map (Redis Cluster protocol,
   Cassandra drivers). Lowest latency; every language client must
   get the protocol right; map updates must be timely.

The map itself — **which shard lives on which node** — is precious
metadata. Typical design: a small **strongly consistent** store
(ZooKeeper, etcd, the database’s own consensus group) as a
**consistent core**. Questions the core must answer:

- Who is allowed to **change** the assignment? One coordinator is
  simple; then that coordinator needs HA without **split-brain
  maps** (two coordinators, two truths — same fencing problem as
  ch. 6).
- How do routers **learn** a move? Watch / push / poll with
  generations so a stale map is obvious.
- During **cutover**, both old and new node may have the data for a
  moment. Reads and writes need a rule (forward, dual-write, block)
  or you silently split the key.

Service discovery (DNS, k8s, Consul) finds **IPs**. Shard routing
finds **owners**. Use DNS for “where is the proxy,” not for “where
is key 42.”

Analytics warehouses also shard, but a query **intentionally**
touches many shards in parallel (ch. 11). OLTP routing is the
opposite instinct: **one shard if you can**.

## Secondary indexes

A primary / partition key tells you which shard owns the row. A
query by `email = …` or `WHERE color = 'red'` does not.

### Local secondary indexes

Each shard indexes **only its own rows**. Writes stay on one shard
(update the row and that shard’s index together). Queries
**scatter** to every shard and merge.

Fine with tens of shards and a **selective** predicate. Painful at
hundreds with an unselective one (“all red cars” when half the
catalog is red): you paid for a fleet and then asked every node.

This is the default mental model of “sharded SQL without magic.”

### Global secondary indexes

The index is sharded by the **secondary value** (`hash(email)`, or
a range of terms). `email = x` goes to **one** (or few) index
shards, which store **postings**: primary keys that match. Fetching
the full records may still hop to other shards.

Writes to one record can update **several** index shards (every
term in a document). Keeping that atomic wants **distributed
transactions** (ch. 8) or you accept **async index lag** (DynamoDB
global secondary indexes — same class of bugs as replication lag
in ch. 6). Cockroach / TiDB / Yugabyte invest here so SQL `WHERE`
still works.

**Search engines** (Elasticsearch, Solr) *are* global inverted
indexes: term → postings list. One term, one shard, cheap. Two
common terms, two huge postings lists on different shards: the
**AND** is a network-heavy intersection. Global indexes shine when
postings are not huge and reads dominate writes.

## How this shows up when you design something

Always say:

- **Shard key** (`user_id` / `tenant_id` is the usual honest default
  for user-centric products). Call out the compound sort key if
  range-within-user matters.
- **Hash vs range**, and which queries become scatter-gather.
- How a **new node** gets data (move shards vs split ranges) and
  whether a human approves rebalance.
- How a request **finds** the shard (smart client, proxy, forward)
  and where the **map** lives.
- What happens to **secondary** lookups (`email`, `geo`, full text):
  local scatter vs global index vs “that is a warehouse query.”
- The **hot key** story (celebrity, flash sale, `ORDER BY created_at`
  on a range-sharded id).

A design that shards by `user_id` and then needs “all unpaid
invoices in the country” has just designed a **batch job** or a
**global index**, whether they know it or not.

Do not shard in an interview (or in week one of a startup) because
the word sounds senior. Name the load parameter that a single
leader cannot take.

## Check yourself

1. Why does `hash(key) % node_count` make adding a node a disaster,
   and what is the usual fix?
2. You shard orders by `customer_id`. How do you serve “orders
   placed in the last hour” for ops?
3. Local vs global secondary index: which hurts writes, which hurts
   reads?
4. A tenant is 40% of QPS. What do you do besides “add more
   shards”?
5. Sequential invoice numbers + range sharding: what happens at
   noon, and what is one key design that avoids it?
6. Automatic rebalancing declares a slow node dead and moves its
   shards. Why can that take the whole cluster down?
7. Why is a Kubernetes Service **not** a shard router?
8. You need `UNIQUE(email)` for the whole product, but you shard by
   `user_id`. What did you just buy: a global index, a txn across
   shards, or a lie?

Continue to [Transactions](../8-transactions/).

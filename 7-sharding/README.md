# 7. Sharding

Companion notes for **Chapter 7** of *Designing Data-Intensive Applications*
(2nd ed.). The first edition called this **partitioning**. Same idea:
when one replica set cannot hold the data or the QPS, you **split** the
dataset across independent groups of nodes. Each piece is a **shard**
(partition). Replication (ch. 6) copies the same data; sharding splits
different data.

## What this chapter is for

Sharding buys **scale** (data size and throughput) and sometimes
**tenancy isolation**. It costs: cross-shard queries, cross-shard
transactions (ch. 8), rebalancing, and a routing layer. Hot spots can
make ten nodes behave like one.

## Pros, cons, and multitenancy

**Pros:** horizontal scale, smaller blast radius (one shard’s compaction
storm is not the whole dataset), possible geo placement of a tenant.

**Cons:** you cannot `JOIN` or `UNIQUE` across the whole dataset without
extra machinery; operations (schema change, backup) multiply; picking a
**shard key** is a one-way-ish decision.

**Multitenancy:** shard by `tenant_id` so one customer’s load stays in
a box you can enlarge or isolate. Ugly when one tenant is 90% of
traffic — they *are* a hot spot; they may need their own shard or a
further split.

## Sharding of key-value data

### By key range

Keys are sorted; shard 1 owns `A–F`, shard 2 `G–M`, … Range scans
(“all orders today if ids are time-sortable”) stay on few shards.
**Hot ranges** are the failure mode: sequential ids, latest timestamp,
a popular prefix. Systems that auto-split hot ranges (HBase,
CockroachDB, Mongo range) exist because humans guess boundaries badly.

This is the classic **key-range partitioning** style.

### By hash of key

`hash(key) % N` or, better, hash onto a ring / a large fixed set of
virtual shards. Load spreads. Range scans become scatter-gather.
**Consistent hashing** and **fixed partition counts** (hash to 1024
slots, map slots to nodes) avoid reshuffling every key when you add a
box — fixed partition counts (hash to many slots, map slots to nodes)
are the usual fix.
Kafka uses a fixed partition count per topic for this reason; changing
it **reshuffles keys** and can break ordering.

### Skew and hot spots

A celebrity `user_id`, a flash-sale `product_id`, a `status = 'new'`
that everyone polls. Hashing does not help if *one key* is the load.
Tricks: add a random suffix and scatter writes (then reads fan out),
cache the hot key, isolate it on dedicated hardware, or change the
product so the hot path is not a single row.

### Rebalancing: automatic vs manual

When you add nodes, some data must move. Automatic rebalance is
convenient and can surprise you (a rebalance during peak, or a
misconfigured cluster that never stops moving). Manual / planned
rebalance is operationally calmer. Either way: move **whole shards**,
not random keys, if you want the key→shard map to stay simple.

## Request routing

The client must hit the node that owns the key. Options:

1. Client knows the map (smart client, Redis cluster protocol).
2. A routing tier looks up the map (mongos, many proxies).
3. Any node accepts and **forwards** (gossip the map).

The map itself is precious metadata: often stored in a small strongly
consistent store (ZooKeeper, etcd — a **consistent core** for cluster
metadata).
If the map is wrong, you silently read stale splits or write to the
wrong owner.

Service discovery and Kubernetes Services
route to *pods*, not to shards. Do not confuse “load balancer” with
“shard router.”

## Secondary indexes

A primary key tells you which shard owns the row. A query by
`email = …` does not.

**Local secondary indexes:** each shard indexes only its own rows.
Writes stay on one shard. Queries **scatter** to every shard and merge.
Fine when you have tens of shards and selective predicates; painful at
hundreds with unselective ones.

**Global secondary indexes:** the index is sharded by the *secondary*
value (`email` hash). The query goes to one (or few) index shards, then
maybe fetches records from other shards. Writes to one record may
update **several** index shards. Keeping that atomic wants distributed
transactions (ch. 8) or you accept **async** index lag (DynamoDB global
indexes). Cockroach / TiDB / Yugabyte invest here so SQL `WHERE`
still works.

Search engines are global inverted indexes in this sense (term →
postings). Intersecting two big postings lists on different shards is
a network-heavy AND.

## How this shows up when you design something

Always say:

- Shard key (`user_id` is the usual honest default for user-centric
 products).
- Hash vs range, and which queries become scatter-gather.
- How a new node gets data.
- How a request finds the shard.
- What happens to secondary lookups (`email`, `geo`, full text).

A design that shards by `user_id` and then needs “all unpaid invoices
in the country” has just designed a batch job or a global index,
whether they know it or not.

## Check yourself

1. Why does `hash(key) % node_count` make adding a node a disaster, and
 what is the usual fix?
2. You shard orders by `customer_id`. How do you serve “orders placed
 in the last hour” for ops?
3. Local vs global secondary index: which hurts writes, which hurts
 reads?
4. A tenant is 40% of QPS. What do you do besides “add more shards”?

Continue to [Transactions](../8-transactions/).

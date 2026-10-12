# 7. Sharding

These notes teach sharding from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What sharding is

Sharding splits a dataset across independent groups of nodes when one replicated group cannot hold the bytes or the write rate. Each piece is a **shard**. The first edition of the book called the same idea partitioning. Normally each record belongs to exactly one shard.

That shard is usually itself replicated: the same record still lives on several machines so one disk failure does not lose it. A single machine often stores many shards, as leader for some and follower for others. The picture is a chessboard of small replicated groups, not three giant copies of the whole database.

Replication copies the same data so a read can move and a failure can be survived. Sharding puts *different* data on different nodes so writes and bytes can grow. Read scale alone does not require sharding. Extra replicas, caches, and better indexes are the first levers for reads. Shard when one leader cannot take the writes, or one node cannot hold the bytes.

The choice of sharding scheme is mostly independent of how each shard replicates. A real design still draws both: who leads each shard, and which keys that shard owns.

Sharding buys scale — data size and write throughput — and sometimes isolation between tenants. It costs cross-shard queries, cross-shard transactions, rebalancing, and a routing layer. A hot spot can make ten nodes behave like one, because every popular key still lands on a single leader.

Vendors do not share a word for the slice:

| You might hear | In |
|---|---|
| Partition | Kafka |
| Range | CockroachDB |
| Region | HBase, TiDB |
| vBucket | Couchbase |
| vnode or token-range | Riak, Cassandra |
| Tablet | Bigtable, Yugabyte, Scylla |

**PostgreSQL partitioning** usually means files on one machine: dropping a partition deletes a file quickly. **Sharding** in these notes means the split is across machines. Some products use both words for the same thing.

A **network partition** is a broken link between nodes. It is not a piece of the keyspace. The word *shard* itself is folklore — Ultima Online’s split worlds, or an old acronym. The idea you need is one of several parallel slices of the data.

## A shard and its replicas

Each shard is a small database with its own leader. A write for a key goes to that shard’s leader, and the followers of that shard replay it.

Picture three nodes and three shards. Node A leads shard 1 and follows shards 2 and 3. Node B leads shard 2 and follows the other two. Node C leads shard 3. A write whose key hashes or ranges into shard 2 goes to node B, and nodes A and C replay that shard’s log as followers. Writes scale when keys spread across shards. A key that everyone hits stays on one leader, and the other nodes do not help that key.

What you gain when the spread is real:

- Horizontal scale: add machines, add capacity, if the load is even.
- A smaller blast radius: one shard’s compaction storm or restore is not the whole dataset.
- A place to pin a tenant or a key range to a geography.
- Operations that are cheap per shard — drop old data, compact, vacuum — stay bounded by one shard’s size.

What you take on:

- A `JOIN` or a `UNIQUE` constraint across the whole dataset needs extra machinery: scatter-gather (ask every shard and merge), a global index, or a transaction manager that can commit on more than one shard.
- Schema change, backup, rolling restart, and “run this backfill” multiply by the shard count.
- The shard key is close to a one-way decision. Changing it rewrites the cluster.
- A cross-shard write can commit on one shard and fail on the other. That partial success is a transaction problem, not a footnote of the router.

## Sharding for many tenants

SaaS products often shard by `tenant_id`, so one customer’s data and load sit in a box you can enlarge, migrate, or isolate.

That placement does three jobs:

- **Noisy-neighbor control.** A heavy tenant does not eat everyone’s buffer pool. Cells or pods of tenants are the same idea at a coarser grain: AWS cell-based architecture groups tenants so one cell’s failure stays inside that cell.
- **Compliance.** Deleting one tenant, or pinning that tenant to an EU region, is a shard operation instead of a scan of the whole table.
- **Schema rollout.** You migrate tenant by tenant instead of one giant `ALTER`. Tenants with requests in flight still need a story; products exist specifically to do transactional schema change across tenant databases.

Three shapes go wrong:

- One tenant is 90% of traffic, or even 40% of queries per second. That tenant *is* the hot spot. More shared shards do not help. That tenant needs its own shard, or several shards of its own.
- Tiny tenants: thousands of almost-empty shards waste connections and file handles. Pack many small tenants onto one shard, and split when one of them grows.
- A query that ignores the tenant — “all unpaid invoices in the country” — becomes scatter-gather or a warehouse job. That is often the right answer. It is not a reason to put the operational store back on one machine.

Citus-style **schema-based sharding** (one Postgres schema per tenant) is this pattern in Postgres vocabulary.

## From a key to a shard

You need a function from key to shard. It has to be cheap to compute, and stable until you intentionally move data.

The key is often **compound**: a **partition key** chooses the shard, and a **sort key** orders rows inside that shard. “All events for this user, in this time range” is then a local range scan on one shard. “All events in this hour, for every user” is not, because the hour is not what chose the shard.

### Ranges of keys

Keys stay sorted. Shard 1 owns `A–F`, shard 2 owns `G–M`, and so on. A range scan — “all orders whose ids fall in today,” when ids sort by time — stays on a few shards. Adjacent keys share a node, which is good for locality and bad when the adjacent keys are all hot.

**Hot ranges** are the failure. Sequential ids, the latest timestamp, and a popular prefix such as `status = new` all pile inserts onto the newest range. Nine other shards sit idle. That is why a UUID primary key combined with range sharding, and a timestamp used as the partition key, both need a second look: the values people insert *now* are neighbors, so they share a shard.

Systems that **auto-split** a range that grew too big or too hot — HBase, CockroachDB, MongoDB range sharding, Bigtable tablets — exist because humans guess the boundaries badly. A split is not free. It rewrites files (the same cost family as LSM compaction, merging immutable sorted files) and it often hits the shard that is already overloaded. Plan for the split cost, or the split happens during the incident.

Worked example, sequential invoices. Invoice numbers increase by one, and ranges own contiguous numbers. At noon every insert lands in the highest range, on one leader. A partition key that spreads — a hash of customer id, with the invoice number as the sort key inside that customer’s shard — moves the insert load with the customers. You lose the single range scan of “every invoice issued in the last hour”; that query becomes scatter-gather or a separate reporting path.

### A hash of the key

Hash the partition key, then map the hash onto shards. A good hash turns skewed keys into a flat space: similar strings do not land near each other. The same input always hashes the same. This is not cryptography. MongoDB historically used MD5. Cassandra and Scylla use Murmur3. A language’s `hashCode()` is not safe across processes, versions, or languages. Do not shard on it.

You get even load for point reads. You pay for it: a range scan becomes scatter-gather unless the range sits *inside* one partition key, walking the sort key. “All orders today” with `hash(order_id)` hits every shard.

`hash(key) % node_count` falls apart when you add a node. Almost every key’s remainder changes, so almost all data moves. Systems that keep order inside a partition, such as Kafka, also break that ordering, because keys that shared a partition no longer do.

The fixes all separate “which shard” from “which node”:

1. **A fixed shard count.** Hash into 1024 virtual shards, or 4096, or some other number you choose up front, then map shards onto nodes. Adding a node moves whole shards. It does not recompute the key’s shard. Kafka topic partitions are this pattern: changing the partition count reshuffles keys. Pick a count you can live with.
2. **Consistent hashing.** Picture a ring. Each node owns an arc. Adding a node steals a slice from its neighbors instead of reshuffling every key. Virtual nodes — many arcs per physical machine — average out luck. Cassandra and Scylla look like this, with many token ranges per node. “Consistent” here means a key tends to stay on the same shard when the cluster grows. It does not mean ACID consistency (the transactional all-or-nothing guarantees a database offers for a group of reads and writes).
3. Other algorithms with the same two promises — balanced load, and little key movement when the node count changes — include **rendezvous hashing** (highest random weight: score the key against every node and pick the winner) and **jump consistent hashing**. Some of them move individual keys onto the new node rather than splitting ranges. Which is better depends on whether you would rather move a few big files or many small key ranges.

### One key that is hot

Hashing spreads keys. It does not spread a single key. A celebrity `user_id`, a flash-sale `product_id`, or a `status = 'new'` queue that every worker polls: one partition, one leader, and the rest of the cluster is idle for that traffic.

The workarounds are all imperfect:

- **Isolate** the hot key on its own shard or its own hardware. A range-based system can put one key in a shard by itself.
- **Cache** the hot read path so the shard is not serving the website.
- **Split writes** with a random suffix, `product_id + 00..99`. Writes fan out across many shards. Reads must gather those shards and merge. Track which keys are in this special mode. Viral load is temporary, so you want a way to **unsalt** later and collapse the key back to one shard.
- **Change the product** so the hot path is not one row. Fan-out on write for a celebrity — write the event to each follower’s feed at write time — moves the cost off the single key.

Cloud systems market adaptive capacity and heat management. Read that as “they will try to move load.” It does not replace a key that was chosen so one row stays writable.

Write-hot and read-hot keys need different treatments. A viral post is often read-hot: cache it. A counter everyone increments is write-hot: shard the counter, or accept a single-leader bottleneck on that one key.

### Moving data when the cluster changes

When you add or remove nodes, some data must move. Two schools:

**Move whole shards** between nodes, with a fixed shard count. The key-to-shard map stays put. Only the shard-to-node map changes. Routing caches stay simple. You need enough shards that a new node can receive a fair share without splitting a shard in the middle of the move.

**Split and merge ranges**, whether the ranges are raw keys or hash ranges. This is more elastic, and it pays the split cost above: rewriting files on a shard that may already be busy.

**Automatic rebalance** is convenient and can autoscale. It is also how a **cascading failure** starts:

1. One node is slow.
2. The cluster declares it dead.
3. Its shards move onto everyone else.
4. Those nodes slow down under the extra copy traffic and the extra writes.
5. More nodes are declared dead.

Rebalance is heavy on network and disk, and the shards must keep taking writes during the move. If you are already at the write ceiling, the split cannot keep up.

**Manual, planned rebalance** is slower and calmer. Run it before a known spike — Cyber Monday, a ticket drop — rather than during one. A human in the loop is a circuit breaker on a dangerous robot, not a rejection of automation.

Either way, move whole shards when you can, rather than random keys one by one. The map stays explainable.

## How a request finds the shard

The client must reach a node that owns that key’s shard. A Kubernetes Service, or a generic load balancer, sends traffic to any healthy pod. That is correct for stateless application servers. It is wrong for a sharded store, unless every pod is a router that knows the map.

Three patterns:

1. **Any node, then forward.** The client hits a random replica. If that node does not own the key, it proxies to the owner. Gossip, or a local copy of the map, has to be fresh enough. You pay an extra hop.
2. **A routing tier.** `mongos`, Vitess vtgate, and many proxies are shard-aware. Clients stay simple. The proxy fleet becomes something you scale and debug on its own.
3. **A smart client.** The client holds the map. The Redis Cluster protocol and Cassandra drivers work this way. Latency is the lowest of the three. Every language’s client has to implement the protocol correctly, and map updates have to arrive before the client sends a write to a node that no longer owns the key.

The map — which shard lives on which node — is precious metadata. A typical design puts it in a small **strongly consistent** store (ZooKeeper, etcd, or the database’s own consensus group) and treats that store as a **consistent core**: a small cluster that agrees on one assignment. The core has to answer three questions:

- One coordinator is the simple answer for who may change the assignment. That coordinator then needs high availability without **split-brain maps** — two coordinators, two truths. The same fencing used for a database leader applies: a generation number so a stale coordinator’s map is rejected.
- Routers learn a move by watch, push, or poll, with generations, so a stale map is obvious rather than silently wrong.
- During **cutover**, both the old node and the new node may hold the key for a moment. Reads and writes need a rule — forward, dual-write, or block — or the key silently splits and two nodes accept different updates.

Service discovery (DNS, Kubernetes, Consul) finds IP addresses. Shard routing finds owners. Use DNS for “where is the proxy,” not for “where is key 42,” because an IP does not know which shard it leads this minute.

Analytics warehouses also shard, and a query there intentionally touches many shards in parallel. Operational routing has the opposite instinct: one shard, if the query can be written that way.

## Secondary indexes

The partition key tells you which shard owns the row. A query by `email = …`, or `WHERE color = 'red'`, does not.

### Local secondary indexes

Each shard indexes only its own rows. A write stays on one shard: update the row and that shard’s index together. A query scatters to every shard and merges the answers.

That is fine with tens of shards and a selective predicate. It hurts at hundreds of shards with an unselective predicate — “all red cars,” when half the catalog is red — because you paid for a fleet and then asked every node. This is the default shape of sharded SQL without extra machinery.

### Global secondary indexes

The index is itself sharded by the secondary value: `hash(email)`, or a range of terms. `email = x` goes to one index shard, or a few, which store **postings**: the primary keys that match. Fetching the full records may still hop to other shards, because the posting is a pointer, not the row.

A write of one record can update several index shards — every term in a document. Keeping that atomic wants a distributed transaction. The alternative is **async index lag**: the row commits now, and the index entry appears later. DynamoDB global secondary indexes are this class of bug, the same family as replication lag. CockroachDB, TiDB, and Yugabyte invest in distributed transactions here so a SQL `WHERE` still works.

Search engines (Elasticsearch, Solr) *are* global inverted indexes: term to postings list. One term, one shard, is cheap. Two common terms mean two huge postings lists on different shards, and the `AND` is a network-heavy intersection. Global indexes shine when postings are not huge and reads dominate writes.

`UNIQUE(email)` on a product sharded by `user_id` is one of three things. A global index sharded by email, so the uniqueness check hits one index shard. A transaction that spans the user shard and the email shard. Or a lie: a local unique index that only prevents duplicates inside one user shard, while two shards both store the same email.

## Mechanism recap

- A record lives on one shard. That shard has its own leader and followers. Writes scale only when keys spread; a hot key stays on one leader.
- Range sharding keeps local scans and piles sequential ids, timestamps, and prefixes like `status = new` onto one range. Hash sharding (MD5, Murmur3 — not `hashCode()`) flattens point reads and turns cross-key scans into scatter-gather.
- `hash(key) % node_count` moves almost every key when a node arrives. Fixed virtual shards (1024, 4096), consistent hashing with virtual nodes, rendezvous hashing, and jump consistent hashing keep most keys put. Rebalance moves whole shards, or splits ranges, and an automatic move of a slow node’s data can cascade.
- Clients find the owner through a forwarding node, a shard-aware proxy (`mongos`, vtgate), or a smart client. The assignment lives in a consistent core (ZooKeeper, etcd, or the database’s consensus group) with fencing, generations, and an explicit cutover rule. DNS finds the proxy, not key 42.
- A local secondary index keeps writes on one shard and scatters reads. A global secondary index (and an inverted index in Elasticsearch or Solr) keeps selective reads on one index shard, spreads one document’s write across index shards, and either uses a distributed transaction or accepts async lag. Tenant sharding isolates load, deletion, and region, until one tenant is the hot spot or a query drops the tenant key.

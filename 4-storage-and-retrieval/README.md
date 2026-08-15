# 4. Storage and retrieval

Companion notes for **Chapter 4** of *Designing Data-Intensive Applications*
(2nd ed.). Chapter 3 picked a data model. This chapter opens the box:
what the engine does when you `INSERT` and `SELECT`. You do not need to
implement a storage engine, but you *do* need to know why Cassandra
loves writes, why Postgres loves point lookups, and why a warehouse is
not a product database.

## What this chapter is for

OLTP and OLAP (ch. 1) are not just different users. They are different
**layouts on disk**.

- OLTP: many small reads/writes by key or secondary index, milliseconds.
- OLAP: few huge scans, aggregations, sequential reads of only the
 columns you need.

If you remember one sentence: **the storage engine is a trade-off
between write shape and read shape.** Everything else is detail.

## Storage and indexing for OLTP

Imagine the simplest database: a file of `key,value` lines. A write
appends. A read scans the file from the start and keeps the latest
value for that key. That is already a **log-structured** store. It is
also O(n) to read, so you add an **index**.

An index is extra structure that makes some queries fast at the cost of
writes (you must update the index too) and disk. That trade-off never
goes away.

### Log-structured storage (LSM-trees)

Writes append to an in-memory table (memtable). When it fills, it is
flushed as an immutable sorted file on disk (**SSTable**). Reads check
memory, then the newest files, then older ones. A **bloom filter**
avoids touching files that cannot contain the key. Background
**compaction** merges files, drops tombstones (deletes) and old
versions.

This family includes **LSM-trees**: RocksDB, Cassandra, HBase, Scylla,
LevelDB, Lucene’s index segments. Kafka’s log is a cousin (append-only
segments, compaction as a policy) — the same segmented-log idea as a
write-ahead log that never truncates until retention says so.

**Why people pick it:** high write throughput (sequential I/O), good
compression, natural fit for “time keeps moving.” **What you pay:**
reads may merge several files; compaction is a background tax (latency
spikes, extra disk); range deletes and updates become tombstones until
compacted.

### B-trees

The other school treats disk as **fixed-size pages** (often 4 KB) you
can overwrite. A B-tree keeps keys sorted in a wide shallow tree.
Lookup is a handful of page reads. An update finds the leaf page and
writes it in place (with a WAL so a crash does not leave a torn page).

This is Postgres, InnoDB, most relational OLTP, many document stores’
primary indexes.

**Why people pick it:** predictable point and range reads, mature
tuning, in-place updates without waiting for compaction. **What you
pay:** random I/O on writes, write amplification from whole-page
writes, fragmentation.

### LSM vs B-tree, as a rule of thumb

| | LSM | B-tree |
|---|---|---|
| Writes | Usually faster | Slower, in-place |
| Reads | Can be slower / more variable | Usually faster, tighter tails |
| Space | Compaction and compression help | More predictable; fill factor / vacuum |
| Ops | Watch compaction | Watch bloat, vacuum, page split |

There is no universal winner. Measure *your* read/write mix. Secondary
indexes exist in both worlds; they are extra trees (or extra LSM
structures) from a column value back to the primary key.

**Covering indexes / clustered indexes** store the row (or extra
columns) in the index so a query never visits the heap. Faster reads,
fatter indexes, slower writes. In-memory engines (Redis and friends)
skip disk layout entirely and
still need structures (hash, skiplist, list) — the same “index vs
scan” idea, in RAM.

## Data storage for analytics

Warehouses (Snowflake, BigQuery, Redshift, ClickHouse, DuckDB, …) are
built for **scans**.

**Column-oriented storage:** instead of storing row 1 entirely, then
row 2, you store all of `amount`, then all of `city`, then all of
`ts`. A query `SUM(amount) WHERE city = 'Tehran'` reads two columns,
not the whole row. Same-type values compress extremely well (run-length,
dictionary). This is why OLAP can chew billions of rows and OLTP
column-stores make miserable product databases (updating one row means
touching many column files).

**Cloud warehouses** lean on object storage (ch. 1 disaggregation):
compute clusters come and go; Parquet/ORC files stay in S3-like
stores.

**Vectorized execution and compilation:** once data is in columns, the
CPU can process thousands of values per instruction (SIMD) or JIT-compile
the query. This is why “SQL on a lake” got fast enough to threaten
traditional warehouses.

**Materialized views and data cubes:** pre-aggregate so dashboards do
not scan raw facts every time. Same idea as fan-out-on-write timelines
(ch. 2), for analysts.

HTAP systems try to be both OLTP and OLAP. They exist. Treat them as a
conscious compromise, not a free lunch.

## Multidimensional, full-text, and vector indexes

**R-trees** (and friends) index several numeric dimensions at once:
“restaurants near this lat/lng.” A B-tree on latitude then longitude
cannot answer “within 1 km” well.

**Full-text search** (Lucene, Elasticsearch, OpenSearch): tokenize
text, invert *term → list of documents*. Boolean queries are set
operations on those lists. Ranking (BM25, etc.) is extra. This is a
**derived** system in most architectures: OLTP is the record; search
is rebuilt from it (CDC, ch. 12).

**Vector embeddings** (2nd edition addition): an ML model maps text,
image, or audio to a point in a high-dimensional space (hundreds or
thousands of floats). “Similar meaning” ≈ “nearby vectors” (cosine or
Euclidean). Indexes:

- **Flat** — compare to everything. Exact, slow.
- **IVF** — cluster the space, search some clusters. Faster, approximate.
- **HNSW** — layered proximity graph; greedy walk from coarse to fine.
 Approximate, the usual production default (Faiss, pgvector, …).

Semantic search / RAG stacks are this plus a model. The vector index
is not a system of record. You still need the document store, a
pipeline to embed on write, and a story for stale embeddings when
the model changes.

## How this shows up when you design something

- User profile by id → B-tree OLTP (Postgres) or LSM if write-heavy
 (Cassandra-style).
- “Posts containing these words” → inverted index, not `LIKE '%foo%'`.
- “Similar to this paragraph” → embeddings + HNSW, approximate k-NN.
- “GMV by city by day” → column store / warehouse, not the OLTP primary.
- Redis → cache or specialized derived structure, not the warehouse.

Naming the engine *type* in an interview (“this is an LSM write path,
so compaction is on the ops checklist”) scores higher than naming a
brand.

## Check yourself

1. A workload is 90% writes of new events, rare point reads. LSM or
 B-tree first, and why?
2. Why is a column store a bad primary store for a shopping cart?
3. What does compaction actually delete, and what goes wrong if it
 falls behind?
4. Why is HNSW allowed to miss a nearer neighbor, and when is that
 unacceptable?

Continue to [Encoding and evolution](../5-encoding-and-evolution/).

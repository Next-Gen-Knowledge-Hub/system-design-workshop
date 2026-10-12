# 4. Storage and retrieval

These notes follow Chapter 4 of *Designing Data-Intensive Applications*, 2nd edition, by Kleppmann and Riccomini.

A data model says what a record is. The storage engine says what happens to the bytes when you insert it and when you read it back. Operational and analytical systems differ here as disk layouts, not only as audiences. An OLTP engine does many small reads and writes by primary key or by a secondary index, and it answers in milliseconds. An OLAP engine does a few huge scans, aggregates them, and wants to read only the columns the query named, sequentially. The engine is a bargain between the shape of the writes and the shape of the reads. Cassandra is fast at writes because its layout appends. Postgres is fast at point lookups because its layout can fetch one key in a handful of page reads. A warehouse is a bad product database because its layout spreads one row across many column files.

## A file of lines, then an index

The smallest database is a file of `key,value` lines. A write appends a line. A read scans from the start of the file and keeps the latest line for that key, so an update is just a later line. That is already a **log-structured** store: the file is the log, and the current value is whatever the log says last. The read is O(n) in the number of lines. An **index** is extra structure, built beside the log, that makes some queries fast. Maintaining it costs writes, because every write updates the index too, and it costs disk. That exchange never goes away. Every engine below is a choice of which queries pay the read and which pay the write.

## Log-structured storage

An **LSM-tree** (log-structured merge-tree) keeps the append behavior and fixes the O(n) read.

A write goes into an in-memory sorted table, the **memtable**. When the memtable fills, it is flushed as an immutable sorted file on disk, an **SSTable** (sorted string table). The file is never edited. A later update of the same key lands in a new memtable and, later, a newer SSTable. A delete writes a **tombstone**, a marker that says "this key is gone," which is itself a write.

A read checks the memtable first, then the newest SSTable, then older ones, and it returns the first value it finds. A **bloom filter** sits in front of each file: a small structure that can say "this key is definitely not in here." The read then skips the file. It can also say "maybe," in which case the file is opened. Background **compaction** merges several SSTables into fewer ones, and while merging it drops tombstones and old versions of a key, so the disk does not keep every historical line forever.

```
write(key, value)
    |
    v
memtable (sorted, in memory)
    |
    | flush when full
    v
SSTable files (immutable, sorted; newer files first on read)
    |
    | compaction merges files
    v
fewer SSTables; old versions and tombstones dropped
```

A read of a key walks the memtable, then newer files, then older files. The bloom filter skips a file that cannot contain the key.

This family includes RocksDB, Cassandra, HBase, Scylla, LevelDB, and the index segments inside Lucene. Kafka's log is a cousin: append-only segments, with compaction as a policy you turn on, the same segmented log as a write-ahead log that is not truncated until a retention rule says so. A **write-ahead log** is the append-only record of changes a database keeps so a crash can replay them; the LSM memtable flush is that idea used as the primary store, not only as crash insurance.

Walk one key. The memtable receives `(user:17, "Sara")`, then `(user:42, "Leila")`, then `(user:17, "Sara K")`. In memory the latest value of `user:17` is `Sara K`. The flush writes an SSTable sorted by key: `user:17 -> Sara K`, `user:42 -> Leila`. That file will not be modified. A later delete of `user:17` writes a tombstone into a new memtable. A read of `user:17` finds the tombstone in memory and returns "deleted" without opening the older file. A read of `user:99` misses the memtable, the bloom filter says the SSTable cannot contain it, and the read returns not-found. Compaction eventually merges the tombstone with `Sara K` and drops both, and the key is gone from disk.

People pick this layout for three reasons. Writes are sequential I/O, appending to the memtable and then writing a whole file in order, so throughput stays high. Sorted runs of similar values compress well. New events arrive as time moves forward, and an append matches that arrival. A workload that is 90 percent writes of new events, with rare point reads, is the shape this path is built for.

What you pay: a read may consult several files and merge what it finds, so read latency is more variable. Compaction is a background tax. It uses disk bandwidth, it spikes latency while it runs, and it needs spare disk to hold the files it is merging. Until compaction runs, an update or a range delete is only a tombstone, and readers still have to honor it. If compaction falls behind, old versions and tombstones accumulate, the disk fills, and a miss may open every file whose bloom filter said "maybe."

## B-trees

The other school treats the disk as **fixed-size pages**, often 4 KB, that can be overwritten. A **B-tree** keeps keys sorted in a wide, shallow tree of those pages. A lookup reads the root page, then an internal page, then a leaf: a handful of page reads, not a scan of the file. An update finds the leaf and writes that page in place.

The overwrite is not safe by itself. A crash halfway through a 4 KB write leaves a **torn page**, half new bytes and half old bytes. So the engine appends the change to a write-ahead log first, then overwrites the page. After a crash, replay of the log restores the page to a complete version.

```
WAL append  -->  overwrite the leaf page (often 4 KB)
lookup: root page --> internal page --> leaf page
```

This is the primary index of Postgres, of InnoDB, of most relational OLTP engines, and of many document stores. People pick it because point reads and range reads are predictable: keys are sorted, so a range walks neighboring leaves. The tuning lore is mature. An update is done when the page and the log are written; nothing waits for a compaction job to drop an old version.

What you pay: writes are random I/O, because the leaf for this key can sit anywhere on disk. Changing 30 bytes rewrites the whole 4 KB page, which is **write amplification**. Pages fragment. A **fill factor** leaves a page partly empty so the next insert has room; the empty space is the cost of avoiding an immediate split. A **page split** happens when a full leaf must become two leaves, and the parent must be updated to point at both. **Bloat** is dead row versions and sparse pages that still occupy disk. **Vacuum** is the job, in Postgres, that reclaims those dead versions so the bloat does not grow without bound.

What you lose if the write-ahead log is skipped: a crash tears a page and the tree no longer contains a consistent key. What you lose if vacuum falls behind: the table's disk use climbs with dead rows, and reads wade through them.

## How the two compare

| | LSM-tree | B-tree |
|---|---|---|
| Writes | Usually faster, sequential | Slower, in place |
| Reads | Can be slower, and more variable | Usually faster, with tighter tails |
| Space | Compaction and compression reclaim old versions | More predictable; watch fill factor and vacuum |
| Operations | Watch compaction | Watch bloat, vacuum, and page splits |

There is no winner for every mix. Measure the read/write ratio you actually have. A write-heavy stream of new events favors the LSM path. A point-lookup path that must stay in a tight latency band favors the B-tree.

**Secondary indexes** exist in both. A secondary index maps a column value, such as an email, back to the primary key. It is another B-tree, or another LSM structure, maintained on the same write. Forget to update it and the email lookup returns a key that no longer matches the row.

A **clustered index** stores the row in the index itself, in key order. A **covering index** stores the extra columns a query needs, alongside the key. Either way the read never visits the **heap**, the area where the full rows live apart from the index. The read is faster. The index is fatter, so it takes more disk and more of the write. Every write updates that fat structure.

In-memory engines such as Redis skip the disk layout and still face the same choice. A hash answers "this key" and does not answer a range. A skiplist or a sorted structure answers a range and costs more on the write. A list preserves order and is the wrong tool for a lookup by key. The index-versus-scan trade is the same idea, in RAM: the structure you maintain is the queries you can answer without walking everything.

## Column storage for scans

Warehouses — Snowflake, BigQuery, Redshift, ClickHouse, DuckDB, and others — are built to scan. **Column-oriented storage** stores a column as its own run of values, instead of storing row 1 completely and then row 2.

A row layout on disk looks like this:

```
row1: amount=10, city=Tehran, ts=t1
row2: amount=15, city=Yazd,   ts=t2
row3: amount=12, city=Tehran, ts=t3
```

A column layout stores three runs:

```
amount: 10, 15, 12
city:   Tehran, Yazd, Tehran
ts:     t1, t2, t3
```

`SUM(amount) WHERE city = 'Tehran'` reads the amount run and the city run. It does not read timestamps, and it does not read the rest of a wide row. Values of one type sit next to each other, so they compress. **Run-length** encoding turns `Tehran, Tehran, Tehran` into "Tehran, three times." **Dictionary** encoding replaces the city string with a small integer and stores the string once. That is why an analytical engine can chew through billions of rows: it reads less, and what it reads is dense.

The same layout is a miserable primary store for a product. A shopping-cart update changes one row: one amount, one sku, one quantity. In a column store that update touches the amount file, the sku file, and the quantity file, each in its own place. The write amplification is one row becoming many file updates, which is the opposite of what a cart path wants.

Cloud warehouses lean on the same storage/compute split as other cloud-native systems: the durable bytes live apart from the machines that scan them. Compute clusters come and go. The files, often Parquet or ORC, stay in an object store of the S3 kind. Shutting the cluster down does not take the data with it.

Once a column is a dense array, the CPU can work on it as a batch. **Vectorized execution** applies one instruction to thousands of values at once (SIMD: single instruction, multiple data). The other route is **compilation**: JIT-compile the query into machine code for this particular filter and aggregate. Either one is why SQL over files in a lake became fast enough to threaten a traditional warehouse that had the only copy of that trick.

A **materialized view** or a **data cube** pre-aggregates so a dashboard does not scan the raw facts on every refresh. GMV (gross merchandise value, the sum of order amounts) by city by day can be a small table, updated when a fact arrives, instead of a scan of every order. This is the same move as a fan-out-on-write timeline: do the merge when the write arrives, and let the read fetch a result that was already built. The cube can be rebuilt from the facts. It is not the system of record.

**HTAP** (hybrid transactional/analytical processing) systems try to serve point updates and big scans in one product. They exist. The compromise is conscious: either the scan still disturbs the point path, or the system keeps two layouts and pays the copy between them. You do not get both shapes for the price of one.

What you lose if the cart's primary store is columnar: every add-to-cart rewrites pieces of many column files. What you lose if the dashboard scans the operational rows: it fights checkout for disk, and it reads columns it does not need. What you lose if the cube is treated as authority: a bug in the aggregation stands until someone rebuilds it from the facts.

## Several dimensions, words, and vectors

A B-tree on one column is the wrong shape for some predicates, even in an operational store.

An **R-tree**, and indexes like it, handle several numeric dimensions together. "Restaurants within 1 km of this latitude and longitude" is a region in two dimensions. A B-tree on latitude, with longitude as a tiebreak, can find a latitude band quickly and then has to filter longitude inside that band. It cannot use both dimensions as one "near me" test, so the band it reads is much larger than the circle the user asked for.

**Full-text search**, as in Lucene, Elasticsearch, and OpenSearch, answers "posts containing these words." The text is tokenized into terms. An **inverted index** maps each term to the list of documents that contain it. A boolean query is a set operation on those lists: the AND of "red" and "shoes" is the intersection of the two lists. Ranking, for example BM25, is a separate score on the documents that survive the sets. `LIKE '%foo%'` does not use this structure. A leading wildcard matches any prefix, so a sorted index on the whole string has no range it can jump to, and the engine scans rows. The inverted index already has the document list for `foo`.

Search is almost always a derived copy. The operational database is the system of record. The index is rebuilt from it. CDC (change data capture) reads the database's change log and republishes each insert, update, and delete, so the index can apply the same change. If the pipeline stops, search and the database diverge, and the database is the one that is right.

**Vector embeddings** map text, an image, or audio through a model to a point in a high-dimensional space: a list of hundreds or thousands of floats. "Similar meaning" is implemented as "nearby vectors," under cosine distance or Euclidean distance. Three indexes are the usual menu.

- **Flat** compares the query vector to every stored vector. The nearest neighbor is exact. The cost is a scan of all of them.
- **IVF** (inverted file) clusters the space and searches only some clusters. It is faster, and it is approximate: a true neighbor that landed in a cluster you skipped is missed.
- **HNSW** (hierarchical navigable small world) builds a layered proximity graph. A search walks greedily from a coarse upper layer down to a finer one. It is approximate, and it is the usual production default, in Faiss, pgvector, and similar libraries. The greedy walk can stop at a local neighbor when a nearer one sits on a link the walk never took.

Semantic search, and a RAG stack (retrieval-augmented generation: fetch nearby documents, then hand them to a model), is this index plus a model. The vector index is not a system of record. You still need the document store, a pipeline that embeds on write, and a plan for **stale embeddings**. When the model changes, the old vectors were produced by the old model, so "nearby" no longer means what the new model means, until every document is embedded again.

What you lose with a B-tree for "within 1 km": you read a latitude band and filter, and the band gets worse as the dataset grows. What you lose with `LIKE '%foo%'`: a scan. What you lose with HNSW or IVF when the product needs the true nearest neighbor: a nearer item the walk or the cluster skip never visited. The flat index is the one that does not miss. That miss is unacceptable exactly when the caller needed the exact neighbor, which is the result flat returns and the approximate indexes do not promise.

## Recap

A log of key-value lines appends writes and scans for the latest value, which is O(n), so an index spends disk and write cost to make a chosen query fast.

An LSM write hits a memtable, flushes to an immutable SSTable, and relies on bloom filters and compaction. Compaction drops tombstones and old versions. The family is RocksDB, Cassandra, HBase, Scylla, LevelDB, and Lucene segments; Kafka's segmented log is the cousin that truncates on retention. You get sequential writes, compression, and a natural fit for events arriving over time, including a mix that is 90 percent new-event writes and rare point reads. You pay variable reads and a compaction tax: latency spikes, extra disk, and tombstones that live until the merge catches up.

A B-tree overwrites 4 KB pages in a shallow sorted tree, after appending to a write-ahead log so a crash cannot leave a torn page. Postgres, InnoDB, most relational OLTP, and many document primary indexes work this way. Point and range reads stay predictable. You pay random I/O, write amplification of a whole page, fragmentation, fill factor, page splits, bloat, and vacuum. Secondary indexes, in either family, map a column value back to the primary key. Clustered and covering indexes store the row or the extra columns so the read skips the heap, and they make writes fatter. Redis and its relatives keep the same choice in RAM: hash, skiplist, or list.

A column store keeps amount, city, and timestamp in separate runs, so `SUM(amount) WHERE city = 'Tehran'` reads two columns and compresses them with run-length and dictionary encoding, which is how a scan gets through billions of rows. Updating one shopping-cart row touches many column files, so this is a bad product primary. Cloud warehouses leave Parquet or ORC in object storage and let compute come and go. Vectorized SIMD execution and JIT compilation are why SQL on a lake got fast. A cube or materialized view pre-aggregates, the way a push timeline precomputes a feed. HTAP serves both layouts only by compromising one of them or by paying for the copy.

R-trees answer multi-dimensional "near me" queries that a latitude-then-longitude B-tree cannot bound tightly. Full-text search tokenizes, inverts term to document list, and runs set operations, with BM25 as a rank on top; it is a derived index fed by CDC, not the system of record. Embeddings turn an object into hundreds or thousands of floats. Flat search is exact and slow. IVF and HNSW are faster and can miss a nearer neighbor. The vector index still needs the documents, an embed-on-write pipeline, and a re-embed when the model changes.

The matching choices are: a user profile by id on a B-tree such as Postgres, or on an LSM if the path is write-heavy in the Cassandra style; posts-containing-words on an inverted index; similar-to-this-paragraph on embeddings plus HNSW, as approximate k-NN (k nearest neighbors); GMV by city by day on a column store; Redis as a cache or a specialized derived structure, not as the warehouse.

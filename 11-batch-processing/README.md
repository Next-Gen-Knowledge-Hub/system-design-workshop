# 11. Batch processing

Companion notes for **Chapter 11** of *Designing Data-Intensive Applications*
(2nd ed.). OLTP (ch. 1–8) serves the live request. A huge amount of
valuable work is **offline**: crunch yesterday’s logs, train a model,
rebuild a search index. The input is a **bounded** dataset — it has an
end — so you can rerun the job and get the same answer. The second
edition rewrote this chapter: MapReduce is history as a product, object
stores and dataflow engines are the present — the *ideas* of map,
shuffle, and reduce remain.

## What this chapter is for

Batch is how you build **derived data** (ch. 1) when a few minutes or
hours of delay is fine. It is also how you **recover**: if the transform
was wrong, you fix the code and run it again. Streams (ch. 12) are the
same idea without an end of file.

Users almost never query the batch filesystem. They query a store that
was **loaded** from batch output. That bridge matters for design docs.

## Batch processing with Unix tools

`cat logs | grep error | awk … | sort | uniq -c` is a batch pipeline:
immutable text in, text out, compose with pipes. **Sort then aggregate**
is how you group more data than fits in memory. Distributed batch
systems are this shape with a filesystem that spans machines and a
scheduler that retries tasks.

Unix lessons that survive at petabyte scale:

- **Immutable inputs** — do not mutate the file you are reading.
- **Pure transforms** — same input, same output; reruns are safe.
- **Output replaces output** — write a new path, then atomically publish.
- Side effects (mapper opens a TCP connection to prod Postgres) make
  retries dangerous and debugging miserable.

Custom programs vs pipelines: a single binary that does everything is
harder to reuse; small stages with clear contracts compose. The same
argument reappears as Spark stages and Airflow tasks.

## Batch processing in distributed systems

### Distributed filesystems

**HDFS-style:** large files split into blocks, replicated across nodes,
a namenode / metadata service tracks the tree. Compute was often
**colocated** with data (“bring code to data”) because networks were
the bottleneck. The programming model: list files, read splits in
parallel.

### Object stores

**S3, GCS, Azure Blob** are the cloud-native default: cheap, durable,
high latency per small request, excellent aggregate throughput.
Compute is **disaggregated** (ch. 1): spin a Spark/Flink/BigQuery job
tonight, shut it down tomorrow, data remains. You pay for listing and
small-file anti-patterns; you win on ops simplicity.

Files for analytics are typically **columnar** (Parquet/ORC) — ch. 4.
A **table** is a set of files plus metadata.

### Distributed job orchestration

Two layers people confuse:

| Layer | Examples | Job |
|---|---|---|
| **Cluster scheduler** | YARN, Kubernetes, Mesos, Spark’s manager | Where tasks run, CPU/RAM packing, retry a failed *task* |
| **Workflow orchestrator** | Airflow, Dagster, Prefect, Temporal for some jobs | DAG of *jobs*, calendars, dependencies, “ETL then train then publish” |

A failed task should retry **without** double-applying side effects.
That is why a stage’s output should be a **new directory / snapshot**
(`s3://bucket/events/dt=2026-08-27/run=uuid/`), then an **atomic swap**
of the pointer the next stage reads (metastore update, `_SUCCESS`
marker, Iceberg/Delta commit).

**Catalogs** (Hive metastore, Glue, Iceberg / Delta / Hudi): which files
are “the table,” which snapshot is current, how to time-travel for
reprocessing.

## Batch processing models

### MapReduce

Map over splits → **shuffle** by key → reduce. It taught the industry
to think in shuffles and tolerant retries. It also checkpointed to disk
between stages. You will rarely start a greenfield MR job in 2026; you
will still feel the shuffle in Spark, Flink, and warehouse engines.

### Dataflow engines

Spark, Flink (batch mode), BigQuery, Snowflake, DuckDB at smaller
scale: a **graph of operators**, pipelining, in-memory shuffle when
possible, whole-stage codegen / vectorization (ch. 4). You write
“join these relations”; the optimizer picks broadcast vs partitioned
join, pushdowns, and column pruning.

### Shuffling data

The expensive part: every record with the same key must meet on the
same reducer / partition. Network and disk amplify. **Skew** (one key
holds 40% of rows) recreates the hot-spot problem (ch. 7). Salting keys,
two-phase aggregation, and skew-aware joins are the same family of
tricks as scattering a celebrity key.

### Joins and grouping

Sort-merge and hash joins over partitions turn “events + dimension
table” into “events enriched with country” without a billion OLTP
point lookups. Grouping is shuffle + aggregate. Windowed analytics in
batch are bounded windows over a finite file — easier than stream
windows (ch. 12).

### Query languages and DataFrames

SQL for analysts and warehouse jobs; **DataFrames** / Datasets for ML
feature pipelines (ch. 3). Same engine, different costume. The second
edition treats DataFrames as a first-class batch citizen, not a side
note.

## Batch use cases

### Extract–transform–load

Operational systems → lake / warehouse. Scheduled. ELT (load first,
transform in the warehouse) is the modern cloud shape. The contract is
**freshness** (“hourly”) and **correctness** (“rerunnable”).

### Analytics

Pre-aggregations, dashboards, ad hoc SQL. Do not run this on the OLTP
primary (ch. 1). Column stores and MPP warehouses exist for this load.

### Machine learning

Dump, clean, featurize, train. Embeddings for vector indexes (ch. 4)
are often a batch (or stream) job. Training wants **reproducible**
snapshots of features, not a live race with production writes.

### Serving derived data

The job’s output is **bulk-loaded** into MySQL, Elasticsearch, a KV
store, or a feature store so the product can read at OLTP latency. The
batch job is not user-facing; the **loaded view** is. Rebuild =
recompute files + reload (or blue/green index swap).

That last point is the bridge to ch. 12–13: derived data architecture
starts here, then goes continuous.

## How this shows up when you design something

- “Recommendations” in an interview: nightly batch is a valid v1;
  streaming updates are v2. Say the freshness SLO.
- Backfills: you *will* reprocess. Design output as immutable dates /
  versions, not `UPDATE` in place on the only copy.
- Separate **orchestration failure** (Airflow retry) from **data
  quality** (row counts, checksums, quarantine).
- Do not run heavy analytics on the OLTP primary.

## Check yourself

1. Why is “mapper writes to Postgres” a worse batch design than
   “mapper writes Parquet, later a loader job”?
2. What is a shuffle, and why does key skew kill it?
3. MapReduce vs Spark-style dataflow: what got faster for users, and
   what idea stayed the same?
4. How would you rebuild a search index from a lake if yesterday’s job
   used a buggy tokenizer?
5. Cluster scheduler vs workflow orchestrator: which one knows that
   “train” must wait for “featurize”?
6. Why atomic publish of a snapshot matters when two readers start a
   job mid-write?
7. Object store vs HDFS: what did disaggregation change about when you
   pay for compute?
8. Where do Iceberg/Delta-style catalogs sit in the “immutable outputs”
   story?

Continue to [Stream processing](../12-stream-processing/).

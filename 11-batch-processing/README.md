# 11. Batch processing

Companion notes for **Chapter 11** of *Designing Data-Intensive Applications*
(2nd ed.). OLTP (ch. 1–8) serves the live request. A huge amount of
valuable work is **offline**: crunch yesterday’s logs, train a model,
rebuild a search index. The input is a **bounded** dataset — it has an
end — so you can rerun the job and get the same answer. The second
edition rewrote this chapter: MapReduce is history, object stores and
dataflow engines are the present.

## What this chapter is for

Batch is how you build **derived data** (ch. 1) when a few minutes or
hours of delay is fine. It is also how you **recover**: if the
transform was wrong, you fix the code and run it again. Streams (ch. 12)
are the same idea without an end of file.

## Unix tools as the miniature

`grep | awk | sort | uniq -c` is a batch pipeline: immutable text in,
text out, compose with pipes. **Sort then aggregate** is how you
group more data than fits in memory. Distributed batch systems are
this, with a filesystem that spans machines and a scheduler that
retries tasks.

The Unix lesson that survives at petabyte scale: **immutable inputs,
pure transforms, output that replaces the previous output.** Debugging
is “run it again on a sample.” Side effects (write to a live OLTP
table from a mapper) make reruns dangerous.

## Distributed filesystems and object stores

**HDFS-style:** big files split into blocks, replicated, a namenode /
metadata service. **Object stores** (S3, GCS, …) are the cloud-native
default: cheap, durable, high latency per request, fantastic
throughput in aggregate. Compute is **disaggregated** (ch. 1): a Spark
cluster tonight, nothing tomorrow, data still in the bucket.

Files are typically columnar (Parquet/ORC) for analytics (ch. 4). A
**metadata / catalog** (Hive metastore, Glue, Iceberg/Delta/Hudi) tells
jobs which files are “the table.”

## Job orchestration

Two layers people confuse:

- **Cluster schedulers** (YARN, Kubernetes, Spark’s own manager): where
  tasks run, retries of a failed task, packing CPUs.
- **Workflow orchestrators** (Airflow, Dagster, Temporal for some
  jobs): the DAG of *jobs* (“ETL then train then publish”), calendars,
  dependencies.

A failed task should be retried **without** double-applying side
effects. That is why the output of a batch stage should be a **new
directory / snapshot**, then an atomic swap of the pointer.

## Batch processing models

**MapReduce:** map over splits, shuffle by key, reduce. It taught the
world to think in shuffles. It also wrote to disk between every stage.
You will rarely start a new MR job in 2026; you will still feel the
shuffle in everything else.

**Dataflow engines** (Spark, Flink batch, BigQuery, DuckDB at smaller
scale): a graph of operators, pipelining, in-memory shuffle when
possible, **SQL and DataFrame APIs** so the optimizer plans joins.
You write “join these two tables”; the engine decides broadcast vs
partitioned join.

**Shuffle:** the expensive part. All records with the same key must
meet. Skew (one key has 40% of rows) is the hot-spot problem again
(ch. 7). Salting keys is the same trick as scattering a celebrity.

**Joins and grouping** in batch: sort-merge and hash joins over
partitions. This is how you turn “events + user table” into “events
enriched with country” without doing OLTP point lookups a billion
times.

**Query languages and DataFrames:** SQL for analytics engineers; DataFrames
for ML feature jobs (ch. 3). Same engine, different costume.

## Batch use cases

- **ETL / ELT** — operational systems → lake/warehouse. Scheduled.
- **Analytics** — pre-aggregations and ad hoc SQL.
- **Machine learning** — dump, clean, featurize, train. Embeddings for
  ch. 4 vector indexes are often a batch (or stream) job.
- **Serving derived data** — the job’s output is bulk-loaded into
  MySQL, Elasticsearch, a KV store, so the product can read it at OLTP
  latency. The batch job is not user-facing; the **loaded view** is.

That last point is the bridge to ch. 12–13: users never query HDFS.
They query a store that was **derived**.

## How this shows up when you design something

- “Recommendations” in an interview: nightly batch is a valid v1;
  streaming updates are v2. Say the freshness SLO.
- Backfills: you *will* reprocess. Design output as immutable dates
  (`dt=2026-08-15/`) not `UPDATE` in place.
- Do not run heavy analytics on the OLTP primary (ch. 1).

## Ties to other workshops

- [kafka-workshop](../../kafka-workshop) — often the *input* to batch
  (dump topics to S3) or the *output* (publish derived events).
- Object storage is the lake; k8s is a common place to run the
  compute ([k8s-workshop](../../k8s-workshop)).

## Check yourself

1. Why is “mapper writes to Postgres” a worse batch design than
   “mapper writes Parquet, later a loader job”?
2. What is a shuffle, and why does key skew kill it?
3. MapReduce vs Spark-style dataflow: what got faster, for users?
4. How would you rebuild a search index from a lake if yesterday’s
   job used a buggy tokenizer?

Continue to [Stream processing](../12-stream-processing/).

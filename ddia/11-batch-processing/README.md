# Batch processing

These notes teach batch processing from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What batch processing is

A checkout service answers one request at a time: reserve the item, charge the card, write the order. That live path is OLTP, online transaction processing. A large share of the valuable work is not on that path. It happens later, offline. You count yesterday's logs, train a ranking model, or rebuild a search index. The input is a **bounded** dataset. It has an end. Because it has an end, you can run the job again on the same bytes and get the same answer, as long as the transform itself has not changed.

The result is **derived data**: a dataset computed from data you already stored. A delay of minutes or hours is acceptable. The same job is how you recover. If the transform was wrong, you fix the code and run it again. A stream is the same idea with no end of file. More events keep arriving, so you cannot wait until "the whole input" is present. Batch is the case where you can.

People almost never query the batch filesystem. They query a store that was **loaded** from the batch output: a search index, a warehouse table, a feature store. The job writes files. A later step publishes those files into something the product can read at the latency of a live request.

MapReduce as a product is history. Object stores and dataflow engines are what people run. The ideas of map, shuffle, and reduce are still the shape of the work.

## A pipeline of small programs

Take one day's web-server logs, already closed:

```
2026-08-27T02:14:03Z level=error sku=8841 msg=timeout
2026-08-27T02:14:09Z level=info  sku=12   msg=ok
2026-08-27T02:14:11Z level=error sku=8841 msg=timeout
```

`cat logs | grep error | awk … | sort | uniq -c` is a batch pipeline. Each stage reads text and writes text. The shell composes the stages with pipes. `grep` keeps the error lines. `awk` pulls out the sku. `sort` orders those keys. `uniq -c` counts runs of the same key. The count is the derived data: sku 8841 timed out twice.

**Sort, then aggregate** is how you group more data than fits in memory. An external sort spills sorted runs to disk and merges them. Once the keys are in order, one pass counts each group. You do not need a hash table the size of the input.

A distributed batch system is this shape plus two pieces: a filesystem that spans machines, and a scheduler that retries a task that died.

Four lessons from Unix still hold at petabyte scale.

**Immutable inputs.** Do not mutate the file you are reading. Yesterday's log stays yesterday's log. A rerun starts from the same bytes.

**Pure transforms.** The same input produces the same output. A rerun is safe because the stage does not depend on the clock, on a stray write, or on whatever else production is doing at that moment.

**Output replaces output.** Write a new path, then publish it atomically. Readers move from the old result to the new result when a pointer changes. They do not read a directory that is still being filled.

**Side effects break the rerun.** Suppose the mapper opens a TCP connection to production Postgres and inserts a row for every record. A retry inserts the row again. A crash halfway through leaves a partial load and no clean place to resume. Debugging is miserable because the output is tangled with live traffic. The durable pattern is the opposite: the mapper writes a file such as Parquet, and a later loader job publishes that file. The compute stage stays pure. The load is a separate step with its own contract.

One binary that filters, counts, joins, and loads is harder to reuse. Small stages with clear contracts compose. One stage filters, the next aggregates, a later task loads. The same argument shows up as Spark stages and as Airflow tasks. You replace one stage without rewriting the others.

## Where the files live

### A distributed filesystem

An HDFS-style filesystem splits large files into blocks and replicates those blocks across nodes. A namenode, or another metadata service, tracks the tree: which file, which blocks, which nodes hold them. Compute was often **colocated** with the data. The phrase is "bring the code to the data." The network was the bottleneck, so shipping a terabyte to a distant CPU cost more than running the task on the machine that already stored the block. The programming model is: list the files, then read the splits in parallel.

### An object store

S3, GCS, and Azure Blob are the usual cloud storage. They are cheap and durable. One small request has high latency. Many requests at once have excellent aggregate throughput. Compute is **disaggregated**. Storage and processing are separate services. You start a Spark, Flink, or BigQuery job tonight, shut the computers down tomorrow, and the data remains. You do not keep a cluster running all week merely to hold the files. You do pay for listing, and you pay for the small-file anti-pattern: a huge number of tiny objects makes listing expensive and the job slow to start. What you gain is simpler operations. The files stay. The computers come and go.

Files for analytics are typically **columnar**. Parquet and ORC store the values of one column together, so a query that needs only `sku` and `country` does not read every field of every event. A **table** here is a set of those files plus metadata that says which files belong to the table right now.

### A catalog names the current snapshot

A catalog — the Hive metastore, Glue, Iceberg, Delta, or Hudi — records which files are "the table," which snapshot is current, and how to time-travel to an older snapshot when you reprocess. The object store holds immutable files. The catalog holds the pointer. Reprocessing a bad day means reading the snapshot for that day, rather than guessing which objects were overwritten.

## Who runs the work

Two layers both get called "the scheduler," and they do different jobs.

| Layer | Examples | What it decides |
|---|---|---|
| Cluster scheduler | YARN, Kubernetes, Mesos, Spark's manager | Where a task runs, how CPU and RAM are packed, whether to retry a failed task |
| Workflow orchestrator | Airflow, Dagster, Prefect, Temporal for some jobs | The DAG of jobs, the calendar, and dependencies such as "ETL, then train, then publish" |

The cluster scheduler places a task and retries it if the machine dies. It does not know that training must wait until featurization has finished. The workflow orchestrator owns that dependency.

A failed task must retry **without** applying a side effect twice. That is why a stage writes a new directory or snapshot, not into the only copy:

`s3://bucket/events/dt=2026-08-27/run=<uuid>/`

The next stage does not discover that path by convention alone. An **atomic swap** moves the pointer it reads. The swap is a metastore update, a `_SUCCESS` marker, or an Iceberg or Delta commit. Before the swap, readers still see the previous complete snapshot. After the swap, they see the new one.

Picture two jobs that both start reading `dt=2026-08-27/` while the writer is still uploading files into that same prefix. One reader lists the prefix at 02:10 and sees the morning files. The other lists it at 02:40 and sees morning plus evening. They compute different totals from what was supposed to be one dataset. Publishing by swapping a pointer closes that window. The pointer moves only when the new snapshot is complete, so a reader sees either the old whole or the new whole.

An **orchestration failure** is not a **data-quality** failure. Airflow retrying a dead task is the first. A task that exits zero with the wrong rows is the second. Catch the second with row counts, checksums, and a quarantine path for records that fail the check, before the snapshot is published.

## How the computation is organized

### MapReduce

Map over splits, **shuffle** by key, then reduce.

On the error log, each mapper reads one split and emits `(8841, 1)` for every timeout. The shuffle sends every pair with key `8841` to the same reducer. The reducer sums that group. A mapper that dies is retried on the same immutable split. The output of a finished stage was checkpointed to disk, so a failed reduce did not have to repeat the map, and the job paid for that disk IO between stages.

You will rarely start a greenfield MapReduce job in 2026. The shuffle is still there in Spark, Flink, and warehouse engines.

### Dataflow engines

Spark, Flink in batch mode, BigQuery, Snowflake, and DuckDB at smaller scale run a **graph of operators**. Stages can pipeline, so one operator's output can flow into the next without a disk checkpoint between them. The shuffle stays in memory when the data fits. Whole-stage code generation and vectorization compile a stretch of that graph into tight code that processes a column in batches, instead of interpreting one row through one operator at a time.

You write "join these relations." The optimizer chooses the plan. A **broadcast join** sends a small table, such as a country lookup, to every worker. A **partitioned join** shuffles both sides on the join key so matching rows meet in one place. **Pushdown** applies a filter at the scan, before later stages see the dropped rows. **Column pruning** reads only the columns the query names.

What got faster is the path between stages and the optimizer's choices. What stayed the same is the requirement that rows with the same key meet.

### The shuffle, and what skew does to it

The expensive part is that meeting. Every record with the same key must arrive at the same reducer or partition. The network carries those records. Disk amplifies the cost when the shuffle spills.

**Skew** means one key holds a disproportionate share. One key with 40% of the rows is enough. That reducer becomes a hot spot: one partition does almost all the work, and the job finishes only when that one task finishes. It is the same shape as a celebrity key that every request touches. The other workers sit idle.

The repairs are ways of scattering that celebrity key.

**Salting** appends a random bucket to the hot key, so `8841` becomes `8841#0` through `8841#7` and the shuffle spreads it. A second aggregation strips the salt and adds the buckets back together.

**Two-phase aggregation** reduces locally first. Each worker emits a partial count. The shuffle then carries those partials, not every raw row.

A **skew-aware join** detects the hot key and handles it apart from the rest, so that one key does not own a single reducer.

All three exist because a shuffle is only as balanced as its keys.

### Joins and grouping

A **sort-merge join** sorts both inputs on the join key and walks them in lockstep. A **hash join** builds a table from one side and probes it with the other. Over partitions, either one turns "events plus a dimension table" into "events enriched with country." You do not do a billion point lookups against the OLTP database, one per event, to ask what country this user is in. You join the event files to a snapshot of the dimension.

Grouping is a shuffle plus an aggregate. Same key, same place, then sum or count.

Windowed analytics in a batch job are **bounded windows** over a finite file. The file ends, so each window ends with it. You can see every event that will ever belong to Tuesday before you emit Tuesday's count. A stream window does not have that closed end. Events keep arriving, and you have to decide when to stop waiting.

### SQL and DataFrames

Analysts and warehouse jobs write SQL. Feature pipelines for machine learning often use DataFrames or Datasets. It is the same engine in a different costume. DataFrames are a first-class way to write the batch job, not a side note. A pipeline that groups clicks by user and joins a country table performs the same shuffle and join as the SQL behind a dashboard.

## What the jobs are for

### From operational systems into a lake or warehouse

Operational databases feed a lake or a warehouse on a schedule. That movement is extract, transform, and load. **ELT** reverses the middle: load the files first, then transform them with the warehouse engine. That is the usual shape in the cloud. The contract has two parts. **Freshness** is a promise such as "hourly," and you state it. **Correctness** means the job is rerunnable. A bad hour is recomputed from the immutable input.

### Analytics

Pre-aggregations, dashboards, and ad hoc SQL belong on column stores and on MPP warehouses, massively parallel engines built to scan and aggregate large files. They do not belong on the OLTP primary. A dashboard that scans a week of clicks competes with checkout for the same CPU, the same disk, and the same locks.

### Machine learning

The path is dump, clean, featurize, train. Embeddings that fill a vector index — numeric vectors used to find similar items — are often produced by a batch job, and sometimes by a stream job. Training wants a **reproducible snapshot** of the features. It does not want a live race with production writes, where Tuesday's training set silently includes half of Wednesday. A dated directory, or a catalog snapshot, is what makes "this model was trained on that input" something you can run again.

### A view the product can read

The job's output is **bulk-loaded** into MySQL, Elasticsearch, a key-value store, or a feature store. The product then reads that copy at OLTP latency. The batch job is not on the user-facing path. The **loaded view** is.

A rebuild recomputes the files and reloads them, or swaps a blue index for a green one. The new index is built fully beside the old one. Traffic moves only when the new one is ready. If yesterday's indexing job used a buggy tokenizer, you fix the tokenizer, recompute the index from the lake snapshot, and swap. You do not edit the live index row by row and hope the retry is safe.

You will reprocess. Write outputs as immutable dates and versions. Do not `UPDATE` the only copy in place. A nightly recommendations job is a valid first version. Streaming updates are a later version, when the freshness target requires them. Say the target: "recommendations may be a day old" is a design choice.

Derived data starts as files computed from a bounded input and loaded into a serving store. The same view can later be maintained continuously as events arrive. The contract does not change. The view is a function of the record, and you can build it again.

## Recap

Batch processing computes derived data from an input that has an end. Leave that input immutable, keep each stage pure, and publish a new snapshot by swapping a pointer, so a retry does not apply a side effect twice and a reader never sees a half-written directory. A cluster scheduler places and retries tasks. A workflow orchestrator orders whole jobs, which is why it knows that training waits for featurization. Map, shuffle, and reduce are still the shape of the work, whether intermediate data is checkpointed to disk or pipelined in memory. Skew turns one key into a hot spot on the shuffle. People read a store loaded from the output, and a wrong transform is fixed by computing that output again from the original files.

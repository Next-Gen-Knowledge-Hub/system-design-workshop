# 3. Data models and query languages

Companion notes for **Chapter 3** of *Designing Data-Intensive Applications*
(2nd ed.). Before you pick a database brand, you pick a **model**: how
you slice the world into records, and how you ask questions of it. The
wrong model makes every feature a join-or-a-hack; the right one makes
the common path boring.

## What this chapter is for

Most “SQL vs NoSQL” arguments are actually **data model** arguments.
This chapter walks a spectrum:

- Relational tables (still the default for a reason)
- Documents (JSON-ish trees)
- Graphs (everything might be connected to everything)
- Event sourcing / CQRS
- DataFrames and arrays (the ML/analytics cousins)

Query languages come with the model: SQL, Cypher, SPARQL, Datalog,
GraphQL. Declarative languages let the engine pick an access path;
imperative ones make you pick it. That difference matters when data
grows.

## Relational versus document

The **relational model** puts data in relations (tables) of tuples
(rows) with a schema. It is 50+ years old and still what you want when:

- Records **refer to each other** a lot (users, orders, payments).
- You need **joins**, constraints, and ad hoc query.
- Analytics will happen in SQL (star / snowflake schemas in the
  warehouse).

The **document model** stores a nested structure (JSON, BSON) as the
unit of load/save. It fits when the unit of access is a **self-contained
tree**: a résumé, a product with embedded variants, a blog post with
comments you always load together. You avoid the
**object-relational mismatch** — no assembling a user from six tables
just to render a profile.

The cost is **relationships**. Many-to-one (a city name, a user id) you
can duplicate or store as an id and join in the app. Many-to-many
(students ↔ courses) gets messy: either duplicate and drift, or you
reintroduce joins and you have a relational model in a document trench
coat. Document databases have been growing join-like features for this
reason; relational databases grew JSON columns for the opposite reason.
The market is **converging**. The model still matters for *what is
natural*.

**Normalization** (don’t store the same fact twice) vs
**denormalization** (copy for read speed) is not a religion. OLTP
tends to normalize to keep writes consistent. Read-heavy APIs and
analytics **star schemas** denormalize on purpose: a fat `fact_orders`
table plus small `dim_date`, `dim_store` dimensions. Snowflake schemas
normalize the dimensions further. Warehouse people live here; product
engineers meet it when “the dashboard is wrong.”

**When to use which:** if your document is the API response and
cross-document links are rare, documents are a joy (see
[mongo-wokshop](../../mongo-wokshop)). If the interesting questions
are joins and constraints, start relational. If the interesting
questions are “N hops from this person,” keep reading.

## Graph-like data models

A **property graph**: nodes with properties, edges with types and
properties. “Alice —FOLLOWS→ Bob,” “Invoice —ISSUED_TO→ Company.”
Queries walk paths of unknown length. That is painful in vanilla SQL
(recursive CTEs exist; they are not why people buy Neo4j).

Languages you will see named:

- **Cypher** — pattern matching on property graphs (`MATCH (a)-[:KNOWS]->(b)`).
- **SQL recursive queries** — same idea, more verbose, improving.
- **Triple stores + SPARQL** — subject–predicate–object (RDF, semantic web).
- **Datalog** — recursive logical rules; the academic grandparent, still
  useful for permissions and reachability.
- **GraphQL** — *not* a graph database. It is an API query language
  where the client asks for a tree of fields. The backend may be
  relational. It shines at “give me this nested JSON” and hurts when
  clients ask for unbounded nested graphs (N+1 queries unless you
  batch). Treat it as a **presentation layer**, not a storage model.

Use graphs when the *product question* is connectivity: fraud rings,
recommendations, bill-of-materials, org charts. Do not use them as a
general OLTP store for “we might join someday.”

## Event sourcing and CQRS

Instead of storing “the account balance is 40,” you store **what
happened**: `Deposited(20)`, `Withdrawn(10)`, … The current state is a
fold over the log. That is **event sourcing**.

**CQRS** (command query responsibility segregation) pairs it with the
timeline idea from chapter 2: the write side appends events; the read
side is a **materialized view** built from those events (SQL table,
search index, Redis structure). You can rebuild a read model by
replaying. You cannot efficiently answer “all users in Tehran” from the
raw log — that is what the view is for.

This is the same instinct as a WAL ([distributed-systems-workshop
Write-Ahead Log](../../distributed-systems-workshop/2-replication/README_Log.md))
and as Kafka compacting a changelog ([kafka-workshop](../../kafka-workshop)).
Chapter 12 will make the stream explicit. Here the point is modeling:
some domains (ledgers, orders, collaborative editors) *are* a history.

Trade-offs: audit and rebuild are excellent; “just update the row” is
gone; schema evolution of *old events* is a long-term tax (ch. 5);
GDPR erasure of one user in an immutable log is a chapter 1 problem.

## DataFrames, matrices, and arrays

OLTP thinks in rows. ML and scientific computing think in **big numeric
tables** and **n-dimensional arrays** (tensors). A **DataFrame** (Pandas,
Spark, Polars) is a relation that is comfortable with thousands of
columns, missing values, and running a Python function over a column.
It is the bridge between “this came from a warehouse” and “this is a
training matrix.”

You will not serve user checkout from a DataFrame. You *will* design
pipelines that dump OLTP/CDC into something a training job can chew.
Chapter 4’s column stores and chapter 11’s batch jobs exist partly for
this.

Specialist models the book only nods at: genome similarity search,
double-entry ledgers (TigerBeetle-style), blockchains. The lesson is
the same: **if the access pattern is weird, there is probably a weird
store that fits, and shoehorning Postgres is a choice with a cost.**

## Schema on write vs schema on read

Relational engines usually **enforce a schema on write**. Document
stores often do not. Your application still has a schema — it is just
**implicit on read**. “Schemaless” means “the database will not save
you from a typo in a field name.” Evolution is easier *and* it is easier
to poison the dataset. Encoding formats in chapter 5 are how you make
that evolution explicit when data leaves the process.

## How this shows up when you design something

Ask:

- What is the **unit of read/write**? A document? A row plus joins? A
  path?
- Which questions are **point gets** vs **multi-hop** vs **scans**?
- Do we need a **system of record** that is a log, plus views?
- Is GraphQL an API convenience or are we accidentally designing
  storage around it?

A standard interview move: user/profile as document or row; social graph
as graph or two tables (`follows`); feed as a derived view (ch. 2).
Naming those three models in one product is what this chapter trains.

## Ties to other workshops

- [mongo-wokshop](../../mongo-wokshop/1-setup/DATA_MODEL_AND_DATA_TYPES.md)
  — document model in the flesh.
- [redis-workshop](../../redis-workshop/2-internals/README_Datastructures.md)
  — not a document store; specialized structures as a *derived* model.
- [kafka-workshop](../../kafka-workshop) — the log that event sourcing
  wants when it grows up.

## Check yourself

1. When is denormalizing a city name into every address document fine,
   and when will it bite you?
2. Why is GraphQL not an answer to “we need a graph database”?
3. How would you rebuild a “follower count” read model after a bug if
   you had events vs if you only had the current table?
4. Star schema: what lives in the fact table vs a dimension, for ride
   receipts?

Continue to [Storage and retrieval](../4-storage-and-retrieval/).

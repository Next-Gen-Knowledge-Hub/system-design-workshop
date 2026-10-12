# 3. Data models and query languages

These notes follow Chapter 3 of *Designing Data-Intensive Applications*, 2nd edition, by Kleppmann and Riccomini.

Before a database brand matters, you pick a **model**: which pieces the world is cut into, and which questions those pieces can answer without a hack. The spectrum in this chapter runs from relational tables, through document trees and graphs, to event logs with separate read models, and on to the numeric tables that training jobs want. The query language comes with the model. SQL, Cypher, SPARQL, Datalog, and GraphQL are the ones you will meet. A **declarative** query says what the result should be, and the engine chooses the access path, including a better path later when an index appears. An **imperative** query says which keys to open and in which order, so the path stays the one you wrote even after the data outgrows it. That difference starts to dominate once the tables are large.

## Relational tables

The **relational model** stores data as relations, which are tables of tuples, which are rows, under a schema that names the columns and their types. It is more than 50 years old. It is still the default when three things are true.

Records point at each other all the time: users, orders, payments. The interesting questions are **joins** (assemble a result from several tables), constraints (reject a payment whose user does not exist), and queries nobody wrote in advance. Analytics will also be SQL, often in a star or snowflake shape described below.

A single order is a row in `orders`, a row in `users`, a row in `payments`. "Payments by this user in January" is a join plus a filter. The engine can use an index on user id this year and a different index next year, without the application rewriting the loop. The schema is checked when the row is written, so a payment missing its user id never lands.

What you lose: a screen that always wants one self-contained tree pays a join, or several, on every read. Rendering one profile by assembling it from six tables is the **object-relational mismatch**: the application thinks in nested objects, and the database thinks in flat rows, so every read rebuilds the object by hand.

## Documents

The **document model** stores one nested value, usually JSON or BSON (JSON in a binary encoding), as the unit you load and save. It fits when that unit is a tree you always read together: a resume, a product with its variants embedded, a blog post with the comments that belong on the same page.

A profile that used to be six tables becomes one document:

```json
{
  "user_id": 17,
  "name": "Sara",
  "address": { "city": "Tehran", "street": "Valiasr" },
  "posts": [ { "id": 9, "title": "Hello" } ]
}
```

One read returns the tree the API was going to send. Nothing is reassembled from six tables. If the document is the API response, and links to other documents are rare, this is a comfortable primary store. MongoDB is the common industrial form of that choice.

The cost shows up when records refer to each other.

A **many-to-one** link, such as a city name or a user id, has two encodings. Copy the city name into every address document. Or store the city id and resolve the name in the application, which is a join you now run yourself. A **many-to-many** link, such as students and courses, is worse. Copy the course list into each student and the student list into each course, and an enrollment is two writes that can diverge: the student says they are in the course, the course does not list the student. That divergence is **drift**. Store only the ids and join them, and you are running a relational model inside the document database, with the constraints implemented in application code.

Document databases have grown join-like features for this reason. Relational databases have grown JSON columns for the opposite reason: a row that is mostly references, plus one nested blob. The products are converging. The model still decides which question is the easy one.

What you lose with a copied city name: the city is renamed, and every address document still holds the old string until something finds and rewrites them. What you lose with duplicated enrollment lists: the two copies drift, and there is no constraint that notices. What you lose if you pick documents for a join-shaped product: every new question becomes either drift or a join rebuilt in the application.

## Normalization, stars, and snowflakes

**Normalization** means a fact lives in one place. The city's name is in the city row, and the address stores the city id. A rename is one write, and every reader that joins sees it. Operational stores tend to normalize so a write cannot forget one of several copies.

**Denormalization** means you copy the fact on purpose, because the read is more important than the single write. Read-heavy APIs do this. Warehouses do it as the basic shape of a **star schema**: one fat fact table plus small dimension tables. `fact_orders` holds the measurements, one row per order: an amount, a foreign key to the day, a foreign key to the store. `dim_date` and `dim_store` hold the descriptive attributes, and they are small. A revenue query sums `fact_orders` and joins the two dimensions to label the buckets. A **snowflake schema** normalizes the dimensions further: `dim_store` points at `dim_city` instead of carrying city attributes itself, so the join chain is longer and the city name still lives in one row.

Warehouse work lives in these shapes. Product engineers meet them when a dashboard and the operational database disagree. The mechanism of that disagreement: the load job copied the city name into the fact row on Monday; the store's city changed on Tuesday; the dashboard still sums Monday's copy. Or the join key was wrong, and Tuesday's amount is added under the wrong city. Either way the chart is a faithful reading of a derived table that no longer matches the system of record.

What you lose by normalizing a read-heavy report: every dashboard query joins the dimensions, and a wide report pays that join on every refresh. What you lose by denormalizing: a later change to the source fact does not move the copies, unless a job rewrites them.

## Graphs

A **property graph** stores nodes that have properties, and edges that have a type and properties of their own. Alice —FOLLOWS→ Bob. Invoice —ISSUED_TO→ Company. The product question is a path whose length you do not know in advance: friends of friends, a fraud ring, a bill of materials, an org chart, a recommendation that walks several hops. Use a graph when connectivity is the question. Using one as a general operational store, on the theory that a join might show up someday, pays graph-write cost for questions a table would have answered directly.

The same walk is painful in ordinary SQL. A **recursive CTE** is a SQL query that refers to its own result, so it can follow a chain. Recursive queries exist, they are getting less awkward, and they are not the reason people buy Neo4j. The languages that match the model:

- **Cypher** matches a pattern on a property graph. `MATCH (a)-[:KNOWS]->(b)` says "bind a and b to nodes with a KNOWS edge between them," and the engine walks those edges.
- **SQL recursive queries** express the same walk in more words.
- **Triple stores** store subject–predicate–object triples, the RDF format that came out of the semantic web. **SPARQL** queries those triples.
- **Datalog** states recursive logical rules. It is the academic ancestor of this family, and it is still a practical way to write permissions and reachability: "X can read Y if X is an admin of a group that can read Y," applied until no new facts appear.
- **GraphQL** is an API language. The client asks for a tree of fields. The server can fill that tree from relational tables. It is a presentation layer. It is useful when the client wants one nested JSON document and wants to name the fields. It fails when the client asks for an unbounded nested graph: the server issues one query for the root and then one query per child, an **N+1**, unless those child lookups are batched into a single query. Nothing in GraphQL stores edges or walks them.

What you lose if the path query is written as a pile of application loops: each extra hop is another round trip, and "how many hops is fraud?" becomes a code change. What you lose if GraphQL is treated as the storage model: the schema of the API becomes the schema of the disk, the N+1 shows up as soon as a client nests freely, and you still do not have a place that can answer "everyone reachable from Alice."

## Event sourcing and CQRS

**Event sourcing** stores what happened, and treats the current row as a result you can recompute. The account is not stored as "the balance is 40." It is stored as events: `Deposited(20)`, then `Deposited(30)`, then `Withdrawn(10)`. The current state is a **fold** over that log: start at 0, apply each event in order, and the sum is 40. The fold can be run again tomorrow and it will get 40 again, as long as the log is still there.

**CQRS** (command query responsibility segregation) splits the write from the read. The write side only appends events. The read side is a **materialized view**, a table or index built from those events and shaped for a particular question: a SQL row `account_id, balance`, a search index, a Redis structure. "What is the balance?" reads the view. "What is the history?" reads the log. You can rebuild the view by replaying the log from the start, which is how you recover after a bug in the view. You cannot efficiently answer "all users in Tehran" from the raw log, because the log is ordered by time, not by city. The view exists so that question is a lookup or a scan of something already organized by city.

Two neighboring mechanisms are easy to mix up with this one, and they are different choices.

A **write-ahead log** appends a description of a change before the database applies it to its pages. If the process crashes mid-apply, it replays the log and finishes. The log is a crash-recovery tool. It is usually truncated once the pages are safe. Event sourcing keeps the log as the system of record.

**Compaction** of a changelog, as Kafka does, keeps the latest value for each key and drops the older values for that key. After compaction, the balance 40 remains and the deposits that produced it are gone. That is the right shape for a table of current values. It is the wrong shape if the product needs the history.

The domain that fits event sourcing is one that already is a history: a ledger, an order's life cycle, a collaborative editor. Audit is then "read the log." A bad view is repaired by replay. The ability to overwrite one row and forget how it got there is gone. Old events stay forever, so a schema change has to give a meaning to events written years ago that lack the new field. Erasure of one person under a law like the GDPR collides with the same immutability: the events that mention them are still in the log unless you rewrite it or destroy the key that can decrypt them.

What you lose if you only store the balance 40: a wrong balance cannot be rebuilt, because the deposits are gone. What you lose if you only store the log: "users in Tehran" scans every event ever written. What you lose if you compact the log and also promised an audit: the fold inputs have been thrown away.

## DataFrames, arrays, and odd access patterns

Operational systems think in rows: one order, one update. Machine learning and scientific computing think in large numeric tables and in **n-dimensional arrays**, also called tensors. A **DataFrame**, as in Pandas, Spark, or Polars, is a relation that is comfortable with thousands of columns, with missing values, and with running a Python function down a column. It is the bridge from "this file came out of a warehouse" to "this is a training matrix."

You do not serve checkout from a DataFrame. You do build a pipeline that copies operational rows, often by tailing the database change log (CDC: each insert, update, and delete republished), into a form a training job can scan. A **column store** lays each column in its own contiguous run, so the job reads the columns it needs and skips the rest. A **batch job** is one finite pass over that dataset: read the columns, produce the matrix, stop. Those two exist in large part so this handoff is possible.

A few access patterns are odd enough that the chapter only points at them: genome similarity search, double-entry ledgers in the style of TigerBeetle, blockchains. The lesson is the same in each case. If the access pattern is strange, a store built for that pattern probably exists. Putting it in Postgres anyway is a real choice: you give up the access path the specialist store was built around, and you pay with scans or with application code that reimplements the rules, such as "every credit has a matching debit."

What you lose if checkout reads the DataFrame: the frame is a batch snapshot, not a row you can lock and update in milliseconds. What you lose if a ledger is a single balance column: you can no longer show that the books balance, because the matching entries were never stored.

## Schema on write and schema on read

Relational engines usually **enforce a schema on write**. The insert of a row that lacks a required column, or that puts a string in an amount column, is rejected before it is stored. Document stores often accept the bytes and leave the check to the reader. The application still has a schema. It is **implicit on read**: the code expects `city`, and it finds out at read time. **Schemaless** means the database will not save you from a typo in a field name. One writer stores `city`. Another stores `cit y`. Both documents sit happily on disk. The reader that asks for `city` silently misses half the rows.

That looseness makes evolution easier, because a new field can be written without a migration that rewrites every document. It also makes it easier to poison the dataset, because old and new shapes accumulate and nothing at write time objects. When the record leaves the process, **encoding** is the set of rules that turn it into bytes and say which fields exist, so the next program can tell a missing field from an unknown one. Without those rules, the typo and the new field look the same: a key some reader does not expect.

What you lose with schema on write: a new field is a migration, and readers that have not been updated must already tolerate its absence. What you lose with schema on read: the bad field name is discovered by a wrong query result, long after the write.

## Recap

A relational table stores rows under a schema, and it earns its place when records reference each other, when you need joins and constraints, and when the questions are not all known ahead of time. A declarative query lets the engine change the access path as the data grows. A document stores one tree — a resume, a product and its variants, a post and its comments — so a profile is not rebuilt from six tables. Many-to-one facts such as a city name can be copied or stored as an id. Many-to-many facts such as students and courses drift if both sides keep a copy, and joining them by id is the relational model again. The products on both sides have been growing the other's feature. The natural question still decides.

Normalization keeps a fact in one place so a write cannot forget a copy. A star schema denormalizes on purpose: fat `fact_orders`, small `dim_date` and `dim_store`. A snowflake schema normalizes the dimensions further. A dashboard goes wrong when the copied city, or the join key, no longer matches the system of record.

A property graph answers path questions: follows, fraud rings, recommendations, bills of materials, org charts. Cypher, recursive SQL, SPARQL over RDF triples, and Datalog are ways to ask those questions. GraphQL asks a server for a tree of fields and can be backed by tables; an unbounded nest becomes an N+1 unless the server batches.

Event sourcing stores `Deposited(20)`, `Deposited(30)`, `Withdrawn(10)` and folds them to the balance 40. CQRS serves reads from a materialized view — a SQL table, a search index, a Redis structure — that can be rebuilt by replay and that can answer "users in Tehran," which the time-ordered log cannot answer efficiently. A write-ahead log is for crash recovery and is truncated. Compaction keeps the latest value per key and drops the history. Old events make schema change and legal erasure a rewrite problem.

A DataFrame (Pandas, Spark, Polars) is the relation you hand to a training job: many columns, missing values, a function down a column. Checkout does not read it. A column store and a batch pass exist so an operational change log can become a matrix. Genome search, double-entry ledgers, and blockchains are the same moral: a strange access path has a store, and forcing it into Postgres spends the path you needed.

Schema on write rejects a bad row at insert. Schema on read discovers a typo when someone queries. Encoding is what makes the field list explicit once the bytes leave the process. One product can use all three models at once: the profile as a document or a row, the follow relationship as a graph or as two tables, and the feed as a derived view built from the others.

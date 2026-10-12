# 6. Pulling apart operational data

These notes teach the operational-data split from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## One database is still one system

A team can split a codebase into services and still be running one system. The knot that remains is the shared operational database: tickets, customers, billing, and inventory in one engine. That engine gives them one transaction, one connection pool, one write-ahead log, one backup window, and one outage. A deploy of any service still waits on one schema migration. The services were a costume. The database was the architecture.

An **architecture quantum** is the smallest piece that can be deployed, scaled, and restored on its own. It includes the running code and the data that code writes. A service is independently deployable only when its data can live and die with it. Two services that share a writable schema are one quantum, however many repositories were opened.

Choosing a data model (tables, documents, or graphs) and choosing how a single store lays bytes on disk (a B-tree, an LSM tree, or a column layout) are questions you ask *inside* one database. The question here is earlier: is this still one store? Split the operational data first. Each domain may then pick a product type. “We chose Postgres” is not evidence that the data can be separated, and a comparison of engine internals does not decide the cut.

Forces that pull data apart are **disintegrators**. Forces that push it back together are **integrators**. Neither column wins by itself. You name both, then pick which pain you will pay. The same fight shows up in a ticketing system, in billing, in a catalog, or in identity. The book’s field-ops example is only a shared label for it.

```
  one operational database
  +---------------------------------------------+
  │ tickets | customers | billing | inventory   │
  │   one txn · one pool · one outage domain    │
  +---------------------------------------------+
            |                         ^
   disintegrators                     integrators
   (change, scale,                    (joins, atomic
    fault, security,                   multi-table writes,
    quantum, type fit)                 “one engine”)
            v                         |
  +----------+  +-----------+  +------------+
  │ tickets  │  │ customers │  │  billing   │
  │ (maybe a │  │ (maybe a  │  │ (maybe a   │
  │  document│  │  relational│  │  NewSQL)   │
  │   store) │  │   store)  │  │            │
  +----------+  +-----------+  +------------+
```

**Polyglot persistence** means more than one operational store *type* in the estate, chosen because the domains differ. You earn that choice after a split. Buying a second database product while the tables still join in the same instance is a second brand, not a second life cycle.

If the integrators dominate, stop. Putting services on top of one shared writable database is a distributed monolith with extra latency.

## Forces that pull data apart

### Change control

Tables do not age together. Customer-profile columns crawl for months. Ticket-state columns churn hourly. One shared schema turns every change into one migration committee. A deploy of service A waits on a migration owned by team B, or both teams edit the same table in the same release train.

The mechanism is a schema each domain can migrate without asking the rest of the company. In a ticketing system, the team that changes ticket state ships that migration on its own clock. The team that changes customer profile does not sit in that review.

What breaks is splitting the services and leaving the tables. The replacement process is “please do not `ALTER` on Friday.” That is a calendar. The data is still one quantum.

### Connection management

Every new service that opens a pool against the monolith database spends a scarce resource: the engine’s max connections, and the memory those sessions hold. You scale the callers and starve the database. Connection limits become the hidden scalability story.

Each domain’s database then has its own pool, sized to that workload. Ticket traffic no longer consumes the sessions billing needs.

A connection pooler such as PgBouncer in front of the *same* instance delays the split. Callers share one life cycle: one restart, one parameter change, one pool that still points at every table.

### Scalability

Hot tables and cold tables that share a buffer pool, a write-ahead log, and a failover story cannot be scaled independently. One domain’s traffic shape dictates hardware for all of them. A read replica of the whole monolith is not “we scaled tickets.” It is a second copy of customers, billing, and inventory as well.

Separate stores let you add capacity where the load is. Tickets can grow disks and replicas without dragging the cold admin tables along.

Scaling the monolith upward forever, then adding a heroic read replica, still copies the tables you did not need. The bottleneck moved; the coupling stayed.

### Fault tolerance

One crash, one restore, or one corrupted catalog page takes down every service that stored facts there. Shared fate is what “one quantum” means in an outage. Availability of “search tickets” is coupled to availability of “post invoices.”

Failure domains should match service domains. A bad migration in billing should not be the outage story for ticket search.

Running the monolith in multiple availability zones is good hygiene for that one database. It is still one blast radius for a bad migration. Both zones run the same schema.

### Security and access

Rows that fall under card-data rules, payroll rules, or health rules sit next to public catalog rows. They inherit the widest permission set someone needed this quarter. Least privilege becomes a grant spreadsheet across fifty services.

Sensitive tables belong behind a store and a network path only the owning service can reach. The catalog service has no route to the card table.

Row-level security inside the shared database is useful for one application. It does not give you an independently deployable boundary. The same engine, the same credentials plane, and the same backup still hold both kinds of rows.

### Database type fit

After a split, a domain might want a document store or a time-series engine because that is how it is read and written. Before a split, “we also run Cassandra” usually means the same facts are written twice. That is the worst of both worlds: two products, one set of truths, and a join-the-dots operational nightmare inside the monolith.

One engine’s sweet spot is forced on every table until the data is separate. Split first, then match the product type to the domain’s access path. The menu is later in this note.

## Forces that push data back together

### Database transactions

Inside one engine, a multi-table update is cheap: the engine commits every row or aborts every row. Isolation levels, aborts, and a two-phase commit *inside that one product* are how the engine keeps that promise. **Two-phase commit** asks each participant to prepare, then tells them all to commit or all to abort. Inside one database product that can be a sane internal mechanism. The moment two domains have two stores, that comfort is gone. An HTTP call is not a commit.

Keep writes that must not diverge in one database until you have an explicit consistency story for the seam. “Ticket open” and “inventory reserve” are that kind of pair. If you split them, each side commits locally, and closing the gap becomes an application protocol: a single owner that both sides ask to write, or a saga. A **saga** is a multi-step business action in which each owner commits in its own database, and a later failure runs a compensating action instead of one `ROLLBACK`. There is a window where the world is half-updated.

Splitting anyway, then pretending the HTTP handler is `COMMIT`, leaves that window with no compensation and no owner.

### Data relationships

Foreign keys and joins are constraints the engine enforces so the application does not have to. Cross-domain foreign keys are the loudest integrator. After the stores split, those constraints die, and a read that used to be one SQL join has to be replaced on purpose: call the owning service, copy a few columns, cache hot keys, or build a serving view whose product is the combination.

Put tables that are always joined for *writes* in the same domain. A screen that only decorates a read can use one of those later access patterns. Almost every screen being a join is a reason to look at which joins are write-time invariants and which are decoration.

“We’ll join in the API gateway” moves the query planner into a timeout. The gateway becomes the monolith.

### One database type

One backup tool, one on-call rotation, and one query language are a real integrator, especially for a small team. That argues for schemas that are separately owned and still collocated on one product. It does not argue for one shared schema that every service can write.

Five engines before you have five domains multiplies operational work with nothing to show for independence. “Postgres for everything,” used as a veto on *schema* splits, makes the same mistake from the other side. Collocation is shared hardware and a shared product. Common ownership is every service writing the same tables. Those are different.

Name the scores. If relationships and atomic multi-table writes dominate, do not extract services onto a shared writable database.

## Five steps for cutting monolithic data

Do not dump the schema and hope. The steps are ordered so you learn what will break before you cut the wire.

### 1. Analyze and create data domains

The entity-relationship diagram is a hairball. Table names follow an old module, not the business. Group tables by who changes them together and which invariants they protect, not by prefix. A **data domain** is the data half of an architecture quantum: cohesive facts with one life cycle. Code components (intake, assignment, invoicing) are a starting map. Then check the tables. Code seams and table seams often disagree: two packages can look separate while every write still lands in one `tickets` table.

Ask of every table: if this domain is down, which product capability is down? If the answer is “all of them,” you have not found a domain yet.

Domains named after teams — “platform,” “shared” — that collect orphan tables are a junk drawer with a service-level agreement. Nobody can say which capability dies with them, because all of them do.

### 2. Assign tables to domains

Junction tables, audit logs, and status codes that every domain reads do not get to stay communal. Each table gets one owning domain. Shared lookup lists (country codes, reason codes) are an ownership decision you make on purpose: one reference-data owner, or a duplicated code table. Duplicate a code table when the values are truly reference data and drift is cheap. Do not duplicate a balance. A balance has one truth.

“Assigned” on a wiki, while three ORMs can still write the table, is not an assignment. Assignment is a connection string and a permission.

When two domains both believe they own a table, do not skip ahead to the physical move. That is **joint ownership**: two writers, one relation. Resolve who writes before you move files. The ways out are a table split, a new domain for the contested concept, one service delegating the write to the owner, or merging the services back. Those are ownership techniques, and they belong before the cutover.

### 3. Separate the connections

Domains can exist on paper while every service still uses the same connection string and the same migration tool. Give each domain one credential and one connection config *while the data still lives in one instance*. Service A can no longer `SELECT` service B’s tables. Broken queries surface here, cheaply, before you have moved a byte.

Treat a cross-domain table reference in the object-relational map as a build-time or startup-time failure. That check is a **fitness function**: an automated test that fails when a structural decision is violated. This one is the rehearsal.

A “read-only user for everyone,” left in place so reporting keeps working, recreates the shared database with extra steps. Reporting belongs on a replica, a warehouse, or an explicit read pattern — a call, a copied projection, or a serving view — not on production credentials that can see every table.

### 4. Move the schemas

Logical ownership can exist while physical coupling remains: a shared write-ahead log, a shared vacuum, a shared restore. Move each domain to its own schema or its own database on the same server if you must. Move it to its own instance when that operational coupling still hurts. Migrations, backups, and parameter groups become per-domain.

“We moved to another schema named `tickets`” and left `PUBLIC` synonyms so the old joins keep compiling means the move did not happen. The SQL still reaches across.

Plan the cut of referential integrity. Cross-schema foreign keys will die. Either the child can exist without the parent for a while — the two facts converge later, each owner committing locally — or the foreign key was a reason to merge the domains. Find out which before the synonym is the only thing holding the system together.

### 5. Switch over

The dangerous cutovers are dual-running with no end, dual-writing with no end, or a flag day with no rollback. Switch **writes** first for one domain. Keep a temporary read path if you must. Measure. Then drop the old tables. Backfill in a way that can be run again without doubling the rows. One domain at a time.

Forever dual-write “until we’re sure” creates two sources of truth and a reconciliation job that is the real system. Dual-write here is a migration tactic with an end date, not an architecture.

A different bug wears the same name. Writing a fact to the operational table and, in a separate call, to a derived log or a search index, then treating both as sources of truth, is a crash window: one side committed, the other did not. That is a derived-data problem. Do not fold it into this cutover. This cutover ends when the old tables are gone and one domain is the writer.

After the switch, the join you used to write in SQL is gone. Replacing that read is a data-access decision (call the owner, copy columns, cache, or a serving view). It is not a reason to undo the switch in a panic.

A design review can use the steps as a checklist. Six services and one write database means you extracted processes, not quanta: the shared writable schema is still one quantum. A schema migration that requires a company meeting means an integrator (a relationship or a transaction) is unnamed, or step 1 has not been done. A connection-pool incident is evidence for step 3, not only a ticket to raise the proxy limit. A join that “has to stay” means that pair of tables is one domain, or you are volunteering for an explicit read pattern after the split.

## Picking a database type after the split

This is not a tour of how a B-tree differs from an LSM tree. Those are internals of one engine. The question here is: now that this domain is a quantum, which *product type* matches how it is used? Pick from the access path, the relationship shape, and the operations story. A trend slide is not an input.

| Type | A domain that fits | A domain that does not |
|---|---|---|
| Relational | Invariants across several entities; ad hoc query; mature transactions | A blob you always load as one document |
| Key-value | Session, cache-shaped facts, get/put by id | “List me everything that matches these seven filters” |
| Document | One aggregate is the unit of load and save (a work order, a catalog item) | Heavy many-to-many you will fake with arrays forever |
| Column family | Wide, sparse, write-heavy rows keyed for a known access path | Multi-row transactions as the common path |
| Graph | The product question is a path (“who can see this,” “related how”) | Point lookups with an occasional friend list |
| NewSQL | You still want SQL and transactions, but one node is the ceiling | You needed a document model and bought SQL under another name |
| Cloud-native | Operations cost dominates; the managed limits match the access path | You need a feature the service will never grow, and you cannot fork it |
| Time-series | Metrics, sensors, append-by-time; retention is a first-class policy | Customer profiles that get updated in place |

**Relational** remains the default inside a domain that looks like ordinary business transactions: orders, ledgers, assignments with constraints. Splitting services does not require you to leave SQL.

**Key-value** fits when the domain’s API *is* the key. Feature flags, device sessions, and idempotency records are that shape. The moment you need secondary indexes and ranges, you wanted a different type, or a relational table with a primary key.

**Document** fits when a ticket, a work order, or a content object is authored and read as a tree. Schema variation across records is a feature. Invariants that cross aggregates are not. Those push you back toward a relational store, or toward a saga if the aggregates already have different owners.

**Column family** (a wide-column store) fits operational telemetry and sparse user-attribute rows when you designed the partition key on purpose. “Cassandra because scale” is not a design. The wrong key is the wrong life: every query that is not the key becomes a scatter across the cluster.

**Graph** fits when traversal *is* the workload: entitlements, topology, recommendations as a product. Using a graph store as a generic application database because “everything is related” recreates the monolith with worse reporting. Point-get transactions with an occasional friend list do not need it.

**NewSQL** (CockroachDB, Spanner-style systems, TiDB, Yugabyte) is the type you pick when the *domain* is still relational and the *node* is the problem: you want SQL and transactions past a single machine. It is not a way to keep a company-wide shared database without admitting it. A NewSQL cluster that every domain writes, with shared tables, is still one quantum.

**Cloud-native** databases (Aurora, Cosmos DB, DynamoDB-class stores, managed document or key-value services) are an operations type as much as a data model. Failover, backups, and scaling are the product. Fit the domain to the limits — item size, transaction scope, query patterns — or you will invent a second store in six months because a feature you need will never appear and you cannot fork the service.

**Time-series** stores are for observations indexed by time: metrics, sensors, traces. Retention, downsampling, and append-only writes are the design. Do not park ticket rows there because they have a `created_at`. Tickets are updated in place. Time-series engines are built to expire observations, not to be the system of record for a case file.

After the split, every domain may still sit on the monolith engine “for now.” Re-evaluate type per domain against the table. Staying relational is a valid outcome. Changing type is allowed because that quantum can migrate alone. Rewriting a domain into a trendy store to celebrate the split, then spending a year rebuilding joins and transactions the old engine gave you, is the failure. The type has to match the access path you already have.

A request for a new engine has a prior question: is the domain already split, or is this a second copy of the same facts?

## Polyglot databases as a decision

A single engine is a poor fit once two domains are already split and measurably different — graph-shaped entitlements next to a ledger, for example. Introduce a second type where the mismatch is measurable: query pain, write shape, retention, or isolation needs. Write the operations cost into the decision: backup, replay, on-call, and the skill to run it. An architecture decision record is the short note of that context, the choice, and the consequence. The cost belongs there, not in a slide.

A new database per team, adopted as a status symbol, multiplies failure modes without multiplying independence. The multiplication is worse when the new store still dual-writes to the old tables. You have two places that claim to be the truth, and neither deploy is independent.

Four checks have to be true before a second type is earned:

1. The domain already has separate connections and a single writer. Ownership is the write credential and the migration repository, not a name on a diagram.
2. The type matches the dominant access path, not a side feature.
3. You know how other domains will read these facts: a call to the owner, a narrow column copy, a cache with a stated freshness, or a serving view. “They will query our tables” is the shared database again.
4. You have accepted that a transaction across stores is an application protocol, not `BEGIN`. Each store commits locally. The pair converges by a background job, by a request that calls both owners and can compensate, or by an event the second owner consumes.

If those four are not true, collocate on one product with **separate databases**. That is already a win over one monolith schema. Polyglot can wait. “We use Redis, therefore we are distributed” fails the definition: a cache beside a shared operational schema is not a second domain store.

## Recap

The shared operational database is one quantum: one transaction, one pool, one outage. Disintegrators (change rate, connections, scale, fault, security, type fit) pull domains apart. Integrators (atomic multi-table writes, foreign keys, one engine’s operational simplicity) push them together. Five steps cut the knot in order: name data domains, assign each table to one owner, separate credentials while the bytes still share an instance, move schemas, then switch writes one domain at a time and end the dual-write. After that, pick a product type from the access path. A second database type waits until ownership, the dominant access path, the read pattern, and the cross-store protocol are already real.

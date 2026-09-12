# 6. Pulling apart operational data

Companion notes for **Chapter 6** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021). You can split the
code into services and still have **one system**: the shared operational
database. This chapter is the work of cutting *that* knot, then choosing
an engine **per domain** — not a tour of storage internals.

Skip it and you will “go microservices” while every deploy still waits
on one schema migration, one connection pool, and one backup window.
The services were a costume. The database was the architecture.

## See also (do not merge)

Picking a **data model** and a **storage engine** is
[DDIA ch. 3](../../ddia/3-data-models/) and
[DDIA ch. 4](../../ddia/4-storage-and-retrieval/). Those chapters
answer “tables vs documents vs graphs” and “B-tree vs LSM vs columns”
*inside one store*. This chapter answers a prior question: **is this
still one store?** Split first; then each domain may pick a type. Do
not paste engine comparisons into a decomposition design, and do not
treat “we chose Postgres” as proof the operational data is separable.

## The mental model

```
  one operational database
  +---------------------------------------------+
  │ tickets | customers | billing | inventory   │
  │   one txn · one pool · one outage domain    │
  +---------------------------------------------+
            |                         ^
   disintegrators                     integrators
   (change, scale,                    (joins, ACID,
    fault, security,                   “one engine”)
    quantum, type fit)
            v                         |
  +----------+  +-----------+  +------------+
  │ tickets  │  │ customers │  │  billing   │
  │ (maybe a │  │ (maybe a  │  │ (maybe a   │
  │  document│  │  relational   NewSQL)     │
  │   store) │  │   store)  │  │            │
  +----------+  +-----------+  +------------+
```

The sentence to keep: **the database is part of the architecture
quantum** — independently deployable only if its data can live and
die with that service.

Two consequences. First, services that share a writeable schema are
one quantum no matter how many repos you opened. Second, polyglot
persistence is a *decision you earn after a split*, not a slogan you
use to justify buying a second brand while the tables still join in
the same instance.

## Data decomposition drivers

Forces pull the data apart (**disintegrators**) or push it back
together (**integrators**). You do not “win” one column. You name both
and pick which pain you will pay. The book’s ticketing/field-ops case
is just a shared example label; the fight is the same in billing,
catalog, or identity.

### Disintegrators

**Change control.** Tables do not age together. Customer profile
columns crawl; ticket state churns hourly. One schema means one
migration committee.

**Problem** — A deploy of service A is blocked on a migration owned
by team B, or worse, both teams edit the same table in the same
release train.
**Solution** — Give each domain a schema it can migrate without
asking the rest of the company.
**Failure mode** — You split services, leave the tables, and invent
“please don’t `ALTER` on Friday” process. That is not decomposition;
it is a calendar.

**Connection management.** Every new service that opens a pool against
the monolith DB spends a scarce resource (max connections, memory for
sessions). You scale *callers* and starve the database.

**Problem** — Connection limits become the hidden scalability story.
**Solution** — Each domain’s database has its own pool, sized to *that*
workload.
**Failure mode** — PgBouncer in front of the same instance. You delayed
the split; you did not create independent life cycles.

**Scalability.** Hot tables and cold tables sharing a buffer pool, a
WAL, and a failover story cannot be scaled independently. Read replicas
of the *whole* monolith are not “we scaled tickets.”

**Problem** — One domain’s traffic shape dictates hardware for all.
**Solution** — Separate stores so you add capacity where the load is.
**Failure mode** — Vertical scale forever, then a heroic read replica
that still replicates the tables you did not need to copy.

**Fault tolerance.** One crash, one restore, one corrupted catalog
page — every service that stored facts there is down. Shared fate is
the definition of one quantum.

**Problem** — Availability of “search tickets” is coupled to
availability of “post invoices.”
**Solution** — Failure domains that match service domains.
**Failure mode** — Multi-AZ on the monolith. Good hygiene, still one
blast radius for a bad migration.

**Security and access.** PCI, payroll, or health rows sitting next to
public catalog rows inherit the widest permission set someone needed
this quarter.

**Problem** — Least privilege is a GRANT spreadsheet across fifty
services.
**Solution** — Put sensitive tables behind a store and a network path
only the owning service can reach.
**Failure mode** — Row-level security as a substitute for ownership.
Useful locally; it does not give you an independently deployable
boundary.

**Database type fit.** After a split, a domain might *want* a document
store or a time-series engine. Before a split, “we also run Cassandra”
usually means dual-writing the same facts — the worst of both worlds.

**Problem** — One engine’s sweet spot is forced on every table.
**Solution** — Split, *then* match type to the domain’s access path
(later in this chapter).
**Failure mode** — Polyglot *inside* the monolith: five products, one
join-the-dots operational nightmare.

### Integrators

**Database transactions.** Inside one engine you get atomic
multi-table updates for cheap. That comfort is
[DDIA ch. 8](../../ddia/8-transactions/) — isolation, aborts, 2PC
*inside a store*. It is an integrator *here* because the moment you
split, you lose that comfort and you will meet
[ch. 9](../9-data-ownership/) and [ch. 12](../12-transactional-sagas/).

**Problem** — “Ticket open” and “inventory reserve” must not diverge.
**Solution** — Keep those writes in **one** database until you have an
explicit consistency story for the seam.
**Failure mode** — Split anyway, then pretend HTTP is `COMMIT`.

**Data relationships.** Foreign keys and joins are not nostalgia.
They are constraints the engine enforces so the application does not.
Cross-domain FKs are the loudest integrator. After the split they
become [distributed access](../10-distributed-data-access/).

**Problem** — Almost every screen is a join.
**Solution** — Put tables that are always joined for *writes* in the
same domain. Read-time joins can move to ch. 10 patterns.
**Failure mode** — “We’ll join in the API gateway.” You moved the
query planner into a timeout.

**One database type (operational simplicity).** One backup tool, one
on-call, one query language. That is a real integrator, especially
for a small team. It argues for *collocated* but **separately owned**
schemas before it argues for one shared schema.

**Problem** — Five engines before you have five domains.
**Solution** — Separate ownership first; stay on one product if the
access patterns still fit.
**Failure mode** — “Postgres for everything” used as a veto on
*schema* splits. Collocation is not common ownership.

Name the scores. If integrators dominate, stop. Extracting services
onto a shared writable database is a distributed monolith with extra
latency.

## Decomposing monolithic data: five steps

Do not “dump the schema and hope.” The steps are ordered so you
learn what will break *before* you cut the wire.

### 1. Analyze and create data domains

**Problem** — The ER diagram is a hairball. Table names follow an
old module, not the business.
**Solution** — Group tables by **who changes them together** and
**which invariants they protect**, not by prefix. A data domain is
the data half of an architecture quantum: cohesive, independently
life-cycled. Use the component domains from
[ch. 5](../5-component-patterns/) as a starting map, then check the
*data* — code seams and table seams often disagree.
**Failure mode** — Domains named after teams (“platform,” “shared”)
that collect orphan tables. That is a junk drawer with a SLA.

Ask of every table: if this domain is down, which product capability
is down? If the answer is “all of them,” you have not found a domain
yet.

### 2. Assign tables to domains

**Problem** — Junction tables, audit logs, and “status” codes that
every domain reads.
**Solution** — Each table gets **one** owning domain. Shared lookup
lists are a later ownership problem ([ch. 9](../9-data-ownership/)),
not an excuse to leave a communal schema. Duplicate a *code table*
if the values are truly reference data and drift is cheap; do not
duplicate a *balance*.
**Failure mode** — “Assigned” on a wiki, still writable from three
ORMs. Assignment is a connection and a permission, not a slide.

When two domains both believe they own a table, you do not skip to
step 5. You are in joint ownership; resolve it *as ownership* before
you physically move files.

### 3. Separate the connections

**Problem** — Even with domains on paper, every service uses the same
connection string and the same migration tool.
**Solution** — One credential and one connection config per domain,
**while the data still lives in one instance**. Service A can no
longer `SELECT` service B’s tables. Broken queries surface here,
cheaply.
**Failure mode** — A “read-only user for everyone” so reporting keeps
working. You just re-created the shared database with extra steps.
Reporting belongs on a replica, a warehouse, or an explicit access
pattern — not on prod credentials.

This step is the rehearsal. Treat violations as build-time or
startup-time failures (fitness: no cross-domain table references in
the ORM map).

### 4. Move the schemas

**Problem** — Logical ownership exists; physical coupling remains
(shared WAL, shared vacuum, shared restore).
**Solution** — Move each domain to its own schema or database
*on the same server* if you must, then to its own instance when
the operational coupling still hurts. Migrations, backups, and
parameter groups become per-domain.
**Failure mode** — “We moved to another schema named `tickets`”
and left `PUBLIC` synonyms so old joins keep compiling. The move
did not happen.

Plan the **cut of referential integrity**. Cross-schema FKs will
die. Either they were a lie (the child can exist without the
parent for a while — eventual consistency) or they were a reason
to merge domains.

### 5. Switch over

**Problem** — Dual-running, dual-writing, or a flag day with no
rollback.
**Solution** — Switch **writes** first for one domain, keep a
temporary read path if you must, measure, then drop the old
tables. Idempotent backfill. One domain at a time.
**Failure mode** — Forever dual-write “until we’re sure.” You now
have two sources of truth and a reconciliation job that is the
real system. Dual-write is a migration tactic with an end date,
not an architecture. (Derived-data dual-write is a different
bug — [DDIA ch. 12](../../ddia/12-stream-processing/) /
[ch. 13](../../ddia/13-streaming-philosophy/); do not merge that
lecture into this cutover.)

After switch-over, the join you used to write in SQL is gone.
That is [ch. 10](../10-distributed-data-access/), not a reason to
undo step 5 in a panic.

## Selecting a database type (after the split)

This is not a recap of LSM vs B-tree. Those are engine internals
([DDIA ch. 4](../../ddia/4-storage-and-retrieval/)). The question
here is: **now that this domain is a quantum, which *product type*
matches how it is used?**

Pick from the access path, the relationship shape, and the
operations story — not from a trend slide.

| Type | A domain that fits | A domain that does not |
|---|---|---|
| Relational | Invariants across several entities; ad hoc query; mature txns | A blob you always load as one document |
| Key-value | Session, cache-shaped facts, get/put by id | “List me everything that matches these seven filters” |
| Document | One aggregate is the unit of load/save (work order, catalog item) | Heavy many-to-many you will fake with arrays forever |
| Column family | Wide, sparse, write-heavy rows keyed for a known access path | Multi-row transactions as the common path |
| Graph | The *product question* is a path (“who can see this,” “related how”) | Point-get OLTP with an occasional friend list |
| NewSQL | You still want SQL + txns, but one node is the ceiling | You needed a document model and bought SQL-by-another-name |
| Cloud-native | Ops cost dominates; the managed limits match the access path | You need a feature the service will never grow (and cannot fork) |
| Time-series | Metrics, sensors, append-by-time, retention is a first-class policy | Customer profiles that get updated in place |

**Relational** remains the default *inside* a domain that looks like
an OLTP business: orders, ledgers, assignments with constraints.
You are not required to flee SQL because you split.

**Key-value** fits when the domain’s API *is* the key. Feature flags,
device sessions, idempotency records. The moment you need secondary
indexes and ranges, you wanted a different type (or a relational
table with a PK).

**Document** fits when the ticket/work-order/content object is
authored and read as a tree. Schema variation across records is a
feature. Cross-aggregate invariants are not — those push you back
toward relational or toward an explicit saga.

**Column family** (wide-column) fits operational telemetry and
sparse user-attribute rows where you designed the partition key
on purpose. It is not “Cassandra because scale.” Wrong key, wrong
life.

**Graph** fits when traversal *is* the workload (entitlements,
topology, recommendations as a product). Using a graph store as a
generic app DB because “everything is related” recreates the
monolith with worse reporting.

**NewSQL** (Cockroach, Spanner-style, TiDB, Yugabyte) is the type
you pick when the *domain* is still relational and the *node* is
the problem. It is not a way to keep a company-wide shared database
without admitting it. Cross-domain NewSQL is still one quantum if
everyone writes the same cluster with shared tables.

**Cloud-native** (Aurora, Cosmos, DynamoDB-class, managed
document/KV) is an *operations* type as much as a data model:
failover, backups, and scaling are the product. Fit the domain to
the *limits* (item size, transaction scope, query patterns) or you
will invent a second store in six months.

**Time-series** is for observations indexed by time: metrics,
IoT, traces. Retention, downsampling, and append-only are the
design. Do not park ticket rows there because they have a
`created_at`.

**Problem** — The split is done, every domain still sits on the
monolith engine “for now.”
**Solution** — Re-evaluate type **per domain** against the table
above. Staying relational is a valid outcome. Changing type is
allowed because the quantum can migrate alone.
**Failure mode** — Rewriting a domain into a trendy store to
celebrate the split, then spending a year rebuilding joins and
transactions the old engine gave you for free.

## Polyglot databases as a decision

Polyglot persistence means **more than one operational store
type in the estate**, chosen because domains differ. It is not
“we use Redis, therefore we are distributed.”

**Problem** — A single engine is a poor fit for two domains you
already split (e.g. graph-shaped entitlements next to a ledger).
**Solution** — Introduce a second type where the *mismatch is
measurable*: query pain, write shape, retention, or isolation
needs. Budget the ops cost (backup, replay, on-call, skill)
explicitly in the ADR.
**Failure mode** — A new database per team as a status symbol.
You multiplied failure modes without multiplying independence —
especially if they still dual-write to the old tables.

A responsible polyglot checklist:

1. The domain already has **separate connections and ownership**.
2. The type matches the **dominant access path**, not a side
   feature.
3. You know how other domains will **read** these facts
   ([ch. 10](../10-distributed-data-access/)).
4. You accepted that **cross-store transactions** are an
   application protocol ([ch. 9](../9-data-ownership/)), not
   `BEGIN`.

If those four are not true, collocate on one product with
**separate databases**. That is already a win over the monolith
schema. Polyglot can wait.

## How this shows up when you design something

- “We extracted six services.” How many **write** databases?
  If the answer is one, you extracted processes, not quanta.
- Schema migration as a company meeting: name the integrator
  (relationships, txns) or start step 1.
- New engine request: is the domain split already, or is this
  a second copy of the same facts?
- Connection pool incidents: treat them as evidence for step 3,
  not only as a proxy config ticket.
- A join that “has to stay”: that pair of tables is one domain,
  or you are volunteering for ch. 10.

## Check yourself

1. A team lists six microservices and one `DATABASE_URL`. Which
   step of the five have they not done, and what failure mode
   are they in? Give a production example.
2. Change control vs connection limits: both are disintegrators.
   How would you tell which one is *hurting you this quarter*?
3. Why is “we run Postgres in Multi-AZ” not an answer to fault
   isolation *between domains*?
4. Foreign keys across what will become two domains: integrator
   or unfinished assignment? What do you do in step 4 when they
   break?
5. Pick a type for a domain whose API is always `GET /session/:id`
   and `PUT` of an opaque blob. Which type is a mismatch, and
   why is that not a DDIA storage-engine question?
6. When is NewSQL the right *domain* choice, and when is it a
   way to postpone splitting tables?
7. Write the four-point polyglot checklist from memory. Which
   point, if skipped, turns polyglot into dual-write?
8. Step 3 (separate connections) while data still shares an
   instance: what bug are you trying to find *before* step 5?
9. A reporting user needs a join across two future domains.
   Why is “leave a shared read credential” a failure mode, and
   what belongs in ch. 10 or the analytical track instead?
10. In one sentence: DDIA 3/4 vs this chapter — engine vs split.
    Then name a design review comment that illegally merges them.

Continue to [Service granularity](../7-service-granularity/).

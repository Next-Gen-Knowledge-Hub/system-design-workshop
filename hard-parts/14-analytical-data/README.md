# 14. Managing analytical data

Companion notes for **Chapter 14** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

Operational services (the product the customer hits) were chapters
6–13: split the data, name an owner, reach it, run a plot, contract
the seam. This chapter is the **other audience**: people and jobs
that scan history — revenue by region, fraud features, "how many
tickets sat overnight." Skip it and you will either crush OLTP with
warehouse queries or build a central lake that every domain waits
on like a second monolith.

## See also (do not merge)

**OLTP vs OLAP as workloads** — point lookups vs scans, why they
hate sharing a disk — is
[DDIA ch. 1](../../ddia/1-architecture-tradeoffs/). **How derived
data is computed** — bounded inputs, object stores, shuffle, reruns
— is [DDIA ch. 11](../../ddia/11-batch-processing/). This folder is
**who owns analytical data and how that ownership couples to the
operational architecture quantum**. Do not rewrite star schemas,
columnar layout, or MapReduce here. If you need "why the warehouse
is a column store," go to DDIA. If you need "why the warehouse
*team* is a bottleneck," stay.

## The mental model

```
  warehouse                         lake
  domains --> [ central ETL ]       domains --> [ dump to files ]
                 |                                  |
                 v                                  v
            [ one model ]                      [ swamp / luck ]
            BI waits here                      nobody owns quality

  mesh
  domain A  --> [ data PRODUCT A ] --ports--> consumers
  domain B  --> [ data PRODUCT B ] --ports--> consumers
                  ^
                  |
           platform (self-serve)
           does not own the facts
```

The one sentence to remember: **analytical data is still data with
an owner**; a platform can host it, but a platform cannot *mean* it.

Two consequences. First, "we have a lake" is not an architecture.
It is a filesystem. Second, mesh is not a storage format. It is an
alignment of **analytical products** with **domains** (and, this
book's punch, with **architecture quanta**). You can fail at mesh
on Snowflake or on Parquet the same way: no owner, no contract, no
SLO.

## Previous approach: the warehouse

**Problem** — Analysts need joins across the business. Operational
databases cannot take the scan. Copies exist anyway (reporting
replicas that still die on Monday morning).

**Solution** — A **central warehouse**: extract from operational
systems, transform into a modeled analytical schema, load into a
store built for scans. One place to query. One semantic layer if
you are disciplined. A BI team (or analytics engineering team)
that speaks SQL for the company. This solved a real problem and
still does, at a certain org size.

What you buy: a single vocabulary ("revenue" has a definition),
access control in one place, tooling that vendors have optimized
for decades, joins that do not require eight HTTP calls.

**Failure** — The warehouse team becomes the **integration
monolith**. Every domain's schema change is an ETL ticket. Every
new question waits in the same queue. Operational teams dump
tables "for later" and walk away; quality is the warehouse's
problem, which means it is nobody's. Coupling: the analytical
estate is one quantum. You cannot release "ticketing analytics"
without negotiating the central model. That coupling is the same
shape as a shared operational database
([ch. 6](../6-operational-data/)), pointed at a different
workload.

The warehouse also tempts **stamp coupling**
([ch. 13](../13-contracts/)) at batch scale: extract `SELECT *`
from every owner because the modelers might need a column someday.
You glued warehouse jobs to operational innards. DDIA will happily
tell you how to make that extract efficient. It will not tell you
you should not have extracted the innards.

## Previous approach: the lake

**Problem** — The warehouse is slow to land new sources (logs,
images, clickstreams, semi-structured dumps). Modeling up front
feels like a waterfall. Storage got cheap.

**Solution** — A **data lake**: land files first (object store,
original or lightly converted formats), **schema-on-read**. Data
scientists and jobs parse what they need. ETL becomes ELT. The
promise is agility: no central modeler between a domain and an
experiment.

What you buy: cheap retention, mixed media, replay
([DDIA 11](../../ddia/11-batch-processing/) likes immutable
inputs), a place for data that is not yet a star schema.

**Failure** — **Swamp.** No owner, no catalog, no quality bar, ten
copies of "customers" with ten meanings. Schema-on-read becomes
schema-on-each-reader: every consumer re-parses a slightly
different mess. The lake team still centralizes **platform**
tickets (buckets, jobs, access) without centralizing **meaning**,
which is the worst of both: a bottleneck that cannot answer
"which file is true?"

Lakes did not fail because files are wrong. They failed because
**landing is not a product**. A file without an SLO, a schema
(even a late one), an owner, and a contract is not analytical
architecture. It is a backup you grep.

Warehouse and lake also share a structural habit: **operational
domains push, a central plane pulls, consumers appear at the end.**
Mesh's bet is that this habit does not scale past a certain number
of domains *as an organization*, regardless of Redshift vs S3.

## Data mesh: definition

**Problem** — You split operational systems so teams could ship
independently, then you **re-centralized the analytical plane**
and recreated the queue. Domain teams do not feel the pain of a
wrong revenue number; a distant platform team does, too late.

**Solution** — **Data mesh** (Dehghani's argument, which this
chapter operationalizes for architects): treat analytical data as
**domain-owned products** on a **self-serve platform** with
**federated governance**. Four moves, one architecture:

1. **Domain ownership** — the people who own ticketing
   operationally also own ticketing *as data others consume*.
   Quality is not a ticket to a warehouse team.
2. **Data as a product** — analytical output has users,
   discoverability, docs, versioning, SLOs, access paths. It is
   not a table you found. It is something you could put a pager
   on.
3. **Self-serve platform** — domains should not each invent
   pipelines, storage, identity, lineage. The platform offers
   paved roads. It does **not** own the facts.
4. **Federated computational governance** — global rules (PII,
   interoperability, naming, compatibility) as **automated
   policy**, not a weekly committee that bottlenecked the old
   warehouse. Local decisions stay local.

Mesh is **socio-technical**. If you buy a "mesh" catalog product
and keep one team filling it from `SELECT *`, you bought a lake
with extra metadata. If you rename the warehouse team "data
product owners" without giving domains engineers, you bought new
vocabulary.

**Failure** — Mesh as a storage migration ("move Snowflake to
mesh"). Mesh as an excuse to dump quality ("each domain for
itself, no shared definitions"). Mesh as a **second** operational
split: every microservice now also runs an ad-hoc Spark job
nobody can replay. Platform-less mesh is twelve snowflakes.
Governance-less mesh is twelve swamps.

## Data product quantum

**Problem** — "Domain owns analytics" is a slide until you can
point at a **unit** that ships, versions, and fails independently.

**Solution** — The **data product quantum**: the smallest
analytical unit that is independently valuable and independently
changeable — the analytical cousin of the **architecture quantum**
in [ch. 2](../2-coupling/).

A product quantum typically has:

- **Input ports** — how facts arrive: CDC, domain events, batch
  extract, another product. Ports are contracts
  ([ch. 13](../13-contracts/)), not "the bucket."
- **Transformation** — the code that makes the product *mean*
  something (clean, conform, aggregate). This is DDIA 11/12
  machinery; here you care that **this code has an owner**.
- **Output ports** — how consumers read: SQL endpoint, files,
  APIs, streams. Multiple ports can be one product if they share
  meaning and release.
- **Operational qualities** — freshness SLO, completeness,
  schema compatibility, discoverability, access control, lineage.

```
  [ operational quantum ]                 [ data product quantum ]
  ticket service + its DB                 "tickets for analytics"
         |                                      ^
         | events / CDC / extract               | output ports
         +--------------------------------------+
                    input port (a contract)
```

If two "products" always release together, share a schema, and
page the same people, they are **one** quantum with two tables.
If a "product" cannot ship because the central modelers have not
approved a column, you do not have a product quantum. You have a
warehouse ticket with extra YAML.

**Failure** — Equating quantum with "a Kafka topic" or "a
Snowflake database." Those are ports and hosts. The quantum is
**meaning + owner + independent change**. Also: one giant
"analytics product" for the company. You rebuilt the warehouse
inside a new noun.

## Coupling to the architecture quantum

**Problem** — Operational and analytical views of the same domain
can glue together until neither can deploy. Or they can drift until
the dashboard is fiction.

**Solution** — Treat the coupling as a design, not a pipeline
accident.

- **Same team, two quanta** is the mesh default: Ticket service
  and "tickets as a product" share a **domain**, not a deployable.
  The service can ship a field the product does not yet expose.
  The product can add an aggregate the service never computes.
  They meet at the **input-port contract**.
- **Same deployable** (service ships, and the only way analytics
  sees data is to read its OLTP tables) means you **did not**
  make an analytical quantum. You made a reporting user on prod.
- **Central extract that reaches into the operational DB** couples
  the warehouse/lake to **schema innards** — stamp coupling at
  3 a.m. CDC of internal tables has the same shape unless you
  treat the CDC stream as a **published** contract, not a private
  WAL leak.
- **Events the domain already meant** ([ch. 11](../11-distributed-workflows/),
  [ch. 13](../13-contracts/)) are a cleaner input port: the
  operational quantum already promised a fact. Analytical
  consumers should still be **derived**, not another writer
  ([DDIA 13](../../ddia/13-streaming-philosophy/) — mention only:
  do not dual-write a second "analytics truth").

Freshness vs isolation is the standing trade-off. Tighter coupling
(read the OLTP DB) is fresher and more dangerous. Looser coupling
(product rebuilt from events, SLO of minutes) is safer and stale.
Name the SLO on the **product**, not on the pipeline vendor.

**Failure** — Operational schema change silently breaks eight
dashboards because the input port *was* the table. The other
failure: a mesh of products so decoupled they **disagree on
"customer"** and the CEO gets three revenues. Federated governance
is the join you refused to centralize as a team. If you also
refuse it as policy, you did not decentralize meaning. You
abandoned it.

## When to use which

**Problem** — Mesh is in the talk track. Warehouses still make
money. Lakes still hold logs. Teams pick a slogan.

**Solution** — Match the **organizational bottleneck**, not the
conference.

Lean **warehouse** (with a lake as cheap landing if you need it)
when: few domains, one analytical consumer plane, a modeling team
that is not yet the longest queue, strong need for a single
semantic layer (finance close). This is still the right default
for a lot of firms.

Lean **lake-plus-warehouse** (lake for raw/replay, warehouse for
certified metrics) when: mixed media and batch recompute matter,
and you can staff **ownership** of curated zones so they do not
swamp. This is an architecture of **tiers**, not a mesh. DDIA 11
is the compute story.

Lean **mesh** when: many domains, many analytical consumers,
warehouse/lake *team* is the constraint, operational teams are
already real architecture quanta, you can staff **product thinking
inside domains**, and you will fund a self-serve platform plus
automated policy. Mesh is expensive. It pays when central modeling
cannot physically keep up.

Do **not** mesh when: you have two teams and one Postgres; when
"domain" is still a slide; when you cannot name a consumer of the
would-be product; when you hoped mesh would replace doing
contracts and SLOs. Do not mesh as a way to skip
[DDIA 1](../../ddia/1-architecture-tradeoffs/)'s OLTP/OLAP split —
you still must not scan last month on the checkout disk.

**Failure** — Replacing a working warehouse with a mesh of twelve
underfunded products because a vendor said "data mesh." The
[ch. 15](../15-trade-off-analysis/) out-of-context trap in a
hoodie.

## How this shows up when you design something

- Draw operational quanta and analytical products as **different
  boxes**. If you cannot find the second box, you are about to
  query prod.
- Name the input-port contract. `SELECT *` is a confession.
- Put a freshness SLO on the product, not on "the pipeline
  usually finishes by 7."
- If two products must agree on a metric, write that as
  governance (shared definition, test) or admit you need a
  certified central model for *that* metric — mesh is not a ban
  on shared numbers.

## Check yourself

1. OLTP vs OLAP is DDIA 1. What extra question does this chapter
   ask that "column store vs row store" does not?
2. Warehouse vs lake: which failure is "one queue for meaning,"
   and which is "no owner of meaning"?
3. Why is a lake without product SLOs not an analytical
   architecture?
4. Name the four mesh moves. Which one, if missing, turns mesh
   into twelve snowflakes? Into twelve swamps?
5. Data product quantum vs architecture quantum: can they share
   a team and still be two quanta? What must sit between them?
6. CDC of private tables into the lake: which coupling from
   [ch. 13](../13-contracts/) did you just buy?
7. A dashboard disagrees with checkout about "open tickets."
   Is that a saga bug, a contract bug, or a product-SLO bug?
   How would you tell?
8. When is a central warehouse still the honest choice? Give an
   org-shaped reason, not a vendor-shaped one.
9. Why does "we moved files to object storage" not implement
   mesh?
10. What would make two analytical tables **one** product
    quantum even if they live in different buckets?

Continue to [Build your own trade-off analysis](../15-trade-off-analysis/).

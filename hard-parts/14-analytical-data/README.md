# 14. Managing analytical data

These notes teach who owns data that is scanned for history, from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## Two audiences, two workloads

The customer-facing services answer point lookups: this ticket, this card, this technician. That workload is **OLTP** (online transaction processing). It wants the row, and it dies if a scan of last month runs on the same disk as checkout.

The other audience scans history: revenue by region, features for fraud, how many tickets sat overnight. That workload is **OLAP** (online analytical processing). Copies appear anyway, including reporting replicas that still fall over on Monday morning, because the operational database cannot take the scan.

This lesson is about who owns that analytical data, and how the ownership couples to the operational **architecture quantum** — the set of parts that ship and fail together. A platform can host the bytes. A platform cannot mean them. Columnar file layouts, star schemas, and batch engines such as MapReduce are a different subject, including how a job bounds its inputs, lands them in an object store, shuffles, and reruns. They explain how a scan runs. They do not explain why a warehouse team becomes the queue. A lake is also where data sits before anyone has built a star schema.

A **contract** is the published shape of a seam: which fields exist, and what they mean. **Stamp coupling** is depending on a composite you did not want, such as `SELECT *` from an owner's table because a column might matter someday. A **system of record** is where a fact is written as truth. A **derived** dataset is computed from that record. Analytical consumers are derived. They are not a second writer of an "analytics truth."

## The warehouse

Analysts need joins across the business. The historical answer is a **central warehouse**: extract from operational systems, transform into one modeled analytical schema, load into a store built for scans. One place to query. One semantic layer — a single vocabulary, so "revenue" has a definition — if the team is disciplined. A business-intelligence or analytics-engineering group speaks SQL for the company. That solved a real problem, and it still does at a certain size of organization.

What you buy: the shared definition, access control in one place, tooling vendors have spent decades on, and joins that do not take eight HTTP calls.

Walk "tickets that sat overnight" through it. Each domain dumps tables. A central extract-transform-load job (ETL) waits on those dumps, reshapes them into the one model, and loads the warehouse. The analyst queries that model. A new column in ticketing is an ETL ticket. A new question waits in the same queue.

The failure mode is the **integration monolith**. The warehouse team is the queue for meaning. Operational teams dump tables "for later" and leave. Quality is "the warehouse's problem," which means it is nobody's. The analytical estate is one quantum: you cannot release ticketing analytics without negotiating the central model. That is the same coupling shape as one shared operational database, pointed at scans instead of point lookups.

The warehouse also stamps at batch scale. `SELECT *` from every owner, because modelers might need a column someday, glues warehouse jobs to operational innards. An efficient extract does not make that glue a good contract.

## The lake

The warehouse is slow to land new sources: logs, images, clickstreams, semi-structured dumps. Modeling up front feels like a waterfall, and storage got cheap. A **data lake** lands files first, in an object store, in the original format or a light conversion, and uses **schema-on-read**: the file has no single model until a job parses it. ETL becomes ELT — extract, load, transform later. Data scientists parse what they need. The promise is agility: no central modeler between a domain and an experiment.

What you buy: cheap retention, mixed media, replay from immutable inputs, and a place for data that is not yet a modeled schema.

The failure mode is the **swamp**. No owner, no catalog, no quality bar, ten copies of "customers" with ten meanings. Schema-on-read becomes schema-on-each-reader: every consumer re-parses a slightly different mess. The lake team still centralizes platform tickets — buckets, jobs, access — and does not centralize meaning. That is a bottleneck that cannot answer "which file is true?"

Lakes did not fail because files are the wrong medium. They failed because landing is not a product. A file without a freshness objective (an **SLO**), a schema even if the schema arrives late, an owner, and a contract is a backup you grep. It is not an analytical architecture. "We have a lake" names a filesystem.

Warehouse and lake share a habit. Operational domains push, a central plane pulls, and consumers appear at the end. The mesh bet is that this habit stops scaling, as an organization, past a certain number of domains. The bet does not depend on Redshift versus S3.

## Data mesh

Operational systems were split so teams could ship. The analytical plane was then re-centralized, and the queue came back. The domain team does not feel a wrong revenue number. A distant platform team feels it, too late.

**Data mesh**, as Dehghani argues it and as architects have to operate it, treats analytical data as domain-owned products on a self-serve platform with federated governance. Four moves, one architecture:

1. **Domain ownership.** The people who own ticketing in production also own ticketing as data other people consume. Quality is not a ticket filed with a warehouse team.
2. **Data as a product.** The output has users, discoverability, docs, versioning, SLOs, and access paths. It is something you could put a pager on. It is not a table someone found.
3. **Self-serve platform.** Domains should not each invent pipelines, storage, identity, and lineage (the record of where a number came from). The platform offers paved roads. It does not own the facts.
4. **Federated computational governance.** Global rules — personal data (PII), interoperability, naming, compatibility — run as automated policy. They are not a weekly committee, which is the bottleneck the warehouse already had. Local decisions stay local.

Mesh is socio-technical: the org and the software move together. A "mesh" catalog product, still filled by one team from `SELECT *`, is a lake with extra metadata. Renaming the warehouse team "data product owners" without giving domains engineers is new vocabulary.

Mesh is not a storage format. You can fail it on Snowflake or on Parquet the same way: no owner, no contract, no SLO. Moving files to object storage implements a lake, or a filesystem. It does not implement the four moves.

The failure modes are specific. Mesh as a storage migration ("move Snowflake to mesh") changes the host and leaves ownership where it was. Mesh as "each domain for itself, no shared definitions" dumps quality. Mesh as a second operational split — every microservice runs an ad-hoc Spark job nobody can replay — multiplies pipelines. Platform-less mesh is twelve snowflakes: each domain invents its own road. Governance-less mesh is twelve swamps: nobody can say which "customer" is the customer.

## The data product quantum

"Domain owns analytics" is a slide until you can point at a unit that ships, versions, and fails independently. The **data product quantum** is that unit: the smallest analytical piece that is independently valuable and independently changeable. It is the analytical cousin of the architecture quantum.

A product quantum typically has four parts.

- **Input ports.** How facts arrive: change-data capture (CDC, a feed of row changes), domain events, a batch extract, or another product. A port is a contract. A bucket is not a port.
- **Transformation.** The code that makes the product mean something: clean, conform, aggregate. That code has an owner. How a batch engine shuffles the bytes is a separate subject.
- **Output ports.** How consumers read: a SQL endpoint, files, APIs, streams. Several ports can be one product when they share meaning and release together.
- **Operational qualities.** Freshness SLO, completeness, schema compatibility, discoverability, access control, lineage.

```
ticket service                         tickets-for-analytics
(operational quantum)                  (data product quantum)
        |                                      ^
        |  TicketOpened, published contract    |
        +--------------------------------------+
                     input port
                            |
                            v
                     SQL output port
                     freshness SLO: 15 minutes
                            |
                            v
                     overnight-tickets dashboard
```

The ticket service can ship a field the product does not yet expose. The product can add an overnight aggregate the service never computes. They meet at the input-port contract. Same team, two quanta, is the mesh default: they share a **domain**, not a deployable.

If two "products" always release together, share a schema, and page the same people, they are one quantum with two tables, even when the files live in different buckets. A Kafka topic is not a quantum. A Snowflake database is not a quantum. Those are ports and hosts. The quantum is meaning, plus an owner, plus independent change. If a "product" cannot ship until central modelers approve a column, it is a warehouse ticket with extra YAML. One giant "analytics product" for the company rebuilds the warehouse inside a new noun.

## How the analytical quantum couples to the operational one

Operational and analytical views of the same domain can glue together until neither can deploy, or drift until the dashboard is fiction. Treat that coupling as a design.

Same deployable — the service ships, and the only way analytics sees data is to read its OLTP tables — means there is no analytical quantum. There is a reporting user on production.

A central extract that reaches into the operational database couples the warehouse or the lake to schema innards. That is stamp coupling at 3 a.m. CDC of private tables has the same shape when the stream is a leak of the database's **write-ahead log** (the private durable record of its writes). Treat the CDC stream as a published contract if it is going to be a port.

Events the domain already meant are a cleaner input port, because the operational quantum already promised the fact. Analytical consumers of those events stay derived. Writing a second analytics truth beside the operational write is a second system of record.

Freshness versus isolation is the standing trade-off. Reading the OLTP database is fresher and more dangerous. Rebuilding the product from events, with an SLO of minutes, is safer and stale. Name the SLO on the product. "The pipeline usually finishes by 7" is not an SLO, and the vendor of the pipeline is not where the SLO lives.

A dashboard that disagrees with checkout about "open tickets" is one of three different bugs, and they do not share a fix.

- The operational plot can be half-done. A **saga** is an application protocol for a plot whose steps commit in different services; "open" can be true in one service and not yet in another. That disagreement lives in the workflow.
- The input port can be wrong. If the port *was* the table, an operational schema change silently breaks eight dashboards. That is a contract bug, the same stamp as `SELECT *`.
- The product can be inside its freshness SLO, or past it. A product rebuilt from events is allowed to lag by the SLO you named. Disagreement inside that window is the SLO. Disagreement because two products each defined "open" or "customer" with no shared rule is abandoned meaning: the CEO gets three revenues.

**Federated governance** is the join you refused to centralize as a team. A shared definition, checked by a test, keeps one metric honest. Mesh does not ban shared numbers. Some metrics, such as a finance close, still want one certified model. Refusing governance as policy does not decentralize meaning. It abandons it.

## Which bottleneck you actually have

Match the organizational bottleneck. A conference slogan is not a bottleneck.

Lean **warehouse**, with a lake as a cheap landing zone if you need one, when there are few domains, one analytical consumer plane, a modeling team that is not yet the longest queue, and a real need for one semantic layer (a finance close is the usual case). That is still an honest default for a lot of firms. The reason is the org, not the vendor.

Lean **lake plus warehouse** when mixed media and batch recompute matter: the lake keeps raw inputs for replay, the warehouse holds certified metrics, and someone owns the curated zones so they do not swamp. That is an architecture of tiers. It is not a mesh.

Lean **mesh** when there are many domains and many analytical consumers, the warehouse or lake *team* is the constraint, operational teams are already real quanta, you can staff product thinking inside domains, and you will fund a self-serve platform plus automated policy. Mesh is expensive. It pays when central modeling cannot physically keep up.

Mesh is the wrong tool when two teams share one Postgres, when "domain" is still a slide, when the would-be product has no named consumer, or when mesh is hoped to replace contracts and SLOs. It also does not cancel the OLTP/OLAP split. Last month's scan still does not belong on the checkout disk.

The out-of-context trap is replacing a working warehouse with twelve underfunded products because a vendor said "data mesh."

On a design, draw the operational quantum and the analytical product as different boxes. If the second box is missing, the design is about to query production. Name the input-port contract. `SELECT *` is a confession. Put a freshness SLO on the product. If two products must agree on a metric, write the shared definition or admit a certified central model for that metric.

## Recap

Analytical data still has an owner. A warehouse gives one vocabulary and becomes one queue for meaning. A lake lands files cheaply and becomes a swamp when landing is not a product. Data mesh aligns analytical products with domains: domain ownership, data as a product, a self-serve platform that does not own the facts, and federated policy. The unit that ships is the data product quantum — input contract, owned transformation, output ports, and an SLO — coupled to the operational quantum on purpose. The host (Snowflake, Parquet, an object store) does not decide which of the three you built.

# 7. Service granularity

Companion notes for **Chapter 7** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021). Chapter 6 asked
whether the **data** can live apart. This chapter asks how **big a
service** should be — not “what a microservice is,” not a line-count
rule. Granularity is a tug-of-war you re-run when a force changes.

Skip it and you will split on slogan (“single responsibility”) until
every user request is a five-hop workflow, *or* you will refuse to
split until a harmless copy change redeploys a payments process.
Both are granularity failures. They just fail on opposite ends.

## The mental model

```
  too coarse                              too fine
  +------------------+                    +----+ +----+ +----+
  │   god service    │                    │ a  │ │ b  │ │ c  │
  │  (one deploy,    │                    │hop │ │hop │ │hop │
  │   many reasons   │                    +----+ +----+ +----+
  │   to change)     │                         chatty request
  +------------------+
          ^                                    ^
          | integrators                        | disintegrators
          |  transactions                      |  scope / function
          |  workflow chatter                  |  code volatility
          |  shared domain code                |  scale / throughput
          |  inseparable data                  |  fault tolerance
          |                                    |  security
          |                                    |  extensibility
          +-------------- balance -------------+
```

The sentence to keep: **granularity is the net of disintegrators
minus integrators**, not a target number of services.

Two consequences. First, two services with the same “entity” name
can be right at different times — load, regulation, or release
rate moved. Second, “we split it because the book’s ticketing
case split something similar” is cargo cult. Steal the *kind of
fight*, not the plot.

When a force is “we need a transaction,” that is this chapter’s
*integrator*, not an isolation-level lecture
([DDIA ch. 8](../../ddia/8-transactions/)). Stay here for “should
these be two processes?” Go there for “what anomalies remain if
they share one DB.”

## Disintegrators

Each of these argues for **smaller** (or at least *separate*)
services. None of them wins alone.

### Service scope and function

**Problem** — One process does two jobs a user would name
differently (“catalog browse” and “settle payment”), so every
change description is a lie: you risked billing to ship a badge
color.
**Solution** — Draw services around **a purpose you can say in
one breath** and a contract other teams can depend on. Purpose
is not “one endpoint” and not “one entity.” A payments service
may own authorize, capture, and refund — one purpose, several
operations.
**Failure mode** — “Single responsibility” as “one function per
repo.” You have created a distributed function call graph. Scope
was supposed to *reduce* reasons to change, not maximize Git
repositories.

Test: if you cannot explain why this service exists without
listing three unrelated nouns, it is too coarse *or* you have
not found its purpose yet. Those are different bugs.

### Code volatility

**Problem** — A volatile corner (pricing rules, integration
adapters, experimental UI BFF) lives in the same deployable as
a stable core (ledger posting, entitlement check). You ship the
core as often as the experiment.
**Solution** — Split along **rate of change**. Stable code
should not ride the release train of fidgety code — not because
deploys are evil, but because each deploy is a chance to break
what was not changing.
**Failure mode** — Splitting by *file type* (“all utilities,”
“all controllers”) instead of by volatility. You will still
redeploy the ledger when a DTO moves. Volatility is a time
series, not a folder.

Fitness you can automate later: measure commit density and
incident density per component; candidates to extract are
outliers, not averages.

### Scalability and throughput

**Problem** — One endpoint is 90% of CPU and the rest is idle
admin. You scale the whole process, including the parts that
do not need replicas — or you cannot scale the hot path
without cloning a giant heap.
**Solution** — Split so the **hot operation** is its own
deployable, with its own capacity, cache, and SLO. Throughput
islands deserve isolation even if the code looks “related.”
**Failure mode** — Splitting a cold path “for symmetry.” You
paid the distributed tax for a function that ran twice an
hour. Scale arguments need a graph, not a feeling.

This is *service* scale. Database scale was [ch. 6](../6-operational-data/).
If the hot path still shares a write schema with the cold path,
you moved the bottleneck one layer down.

### Fault tolerance

**Problem** — A flaky dependency or a leak in a non-critical
feature takes down the process that also serves the critical
path. One OOM, one thread pool, one poison message — shared
fate.
**Solution** — Separate **failure domains**. The thing that
may collapse (image resize, outbound fax, ML ranking) should
not share a JVM/runtime with “place order.”
**Failure mode** — Bulkheads inside one process as a permanent
stand-in for a split you already know you need. Bulkheads are
good; they are not an independent quantum. A bad deploy still
hits both sides.

Ask: if this library wedges the event loop, what else dies?
If the answer includes money or login, you have an integrator
fighting a disintegrator — name both.

### Security

**Problem** — Secrets, PII, or card data share a process,
image, and IAM role with a public feature. The attack surface
is the union. Auditors treat the union as in-scope.
**Solution** — Split so the **trust zone** is small: fewer
endpoints, tighter network policy, separate key material,
separate audit log. PCI-in-a-box next to a marketing CMS is
how scope explodes.
**Failure mode** — “We encrypt the column” while the web
tier still has `SELECT` on the vault. Encryption is not a
granularity boundary. A service boundary with no route from
the public site *is*.

Security splits are expensive (two on-call rotations, two
identity stories). Use them when the threat model or the
regulator pays you back. Do not use them to look serious.

### Extensibility

**Problem** — Every new customer variant is an `if` in the
core service. You cannot add a plugin without shipping the
engine. Roadmap turns into a permutation matrix.
**Solution** — Extract the **variation point**: a rules
service, an extension sidecar, a webhook owner — something
you can deploy without republishing the stable core.
**Failure mode** — A “plugin service” that is actually a
scripting engine in the core with extra steps, or a zoo of
one-off services with no shared contract. Extensibility
without [contracts](../13-contracts/) is just more code.

Extensibility is the disintegrator you feel late: the service
*works*, and then the seventh variant makes the eighth
unshippable.

## Integrators

Each of these argues for **keeping things together** — or for
merging a split you already regret.

### Database transactions

**Problem** — Two operations must commit together or not at
all (reserve inventory and open the ticket; take payment and
record entitlement). In one service and one database, that is
a transaction. In two services, it is a protocol.
**Solution** — If the invariant is **synchronous and local**,
keep one service (and likely one data domain). If you still
split, you are volunteering for
[ownership + eventual consistency](../9-data-ownership/) and
[sagas](../12-transactional-sagas/), not for “a smaller class.”
**Failure mode** — Split, then 2PC across the two databases
as the happy path. That 2PC is
[DDIA ch. 8](../../ddia/8-transactions/); as *architecture*
it couples availability. Granularity lost, coordination tax
kept.

Integrator strength is proportional to how bad a partial
success is. “Email didn’t send” is not “money moved twice.”

### Workflow and choreography

**Problem** — A single user action (register, check out,
assign) becomes a conversation among many services: eight
calls, three timeouts, a compensating path nobody can draw.
**Solution** — If the workflow is **the product**, consider
one service for the duration of that action, or accept an
explicit orchestrator later
([ch. 11](../11-distributed-workflows/)). Chatty
choreography is a smell that granularity went past the user
step.
**Failure mode** — “We will make it async” to hide the
chatter. You hid the hop count from the request path and
moved it into “why is the UI still spinning.” Async is a
tool, not a pardon.

Count **synchronous hops on the critical path**. Each hop is
an SLO you inherited. If you cannot name the hop budget, you
are not done designing.

### Shared code

**Problem** — Two would-be services share a thick domain
module (pricing, tax, entitlement). Split them and you either
copy it ([ch. 8](../8-reuse-patterns/) replication) or extract
a library that couples their deploys, or extract a shared
service that becomes the new monolith.
**Solution** — If the shared code *is* the domain, maybe you
have **one** service. If it is infrastructure (logging, auth
middleware), it is not a granularity argument — take it to
ch. 8. Do not confuse those.
**Failure mode** — `company-commons` as the reason two
unrelated domains cannot release. Shared *domain* code is an
integrator; shared *platform* code is a reuse choice.

### Data relationships

**Problem** — The two services would always join. You just
finished [ch. 6](../6-operational-data/) and the tables still
want to live together.
**Solution** — Collocate the services *or* collocate the
data and accept they are one quantum. Relationships that are
write-time invariants are stronger integrators than
read-time decorations (those can wait for
[ch. 10](../10-distributed-data-access/)).
**Failure mode** — “Entity service” per table (`Customer`,
`Address`, `Email`). You maximized relationship pain. Table
≠ service. That is the classic overshoot.

## Finding the right balance

There is no formula that outputs a service count. There is a
repeatable argument:

1. List disintegrators that **hurt this quarter** (not in
   theory).
2. List integrators that **would fire if you split**.
3. Estimate the **cost of the seam** (latency, ownership,
   on-call, saga).
4. Split (or merge) **one** seam. Measure. Repeat.

**Problem** — The team wants a once-and-for-all map of
services for the next five years.
**Solution** — Treat granularity as an **ADR you expect to
revise**. Fitness: hop count on a user journey, deploy
frequency per service, error-budget sharing.
**Failure mode** — A target architecture slide with 40
boxes, none of which match yesterday’s incidents. Another
failure: refusing to merge. Merge is a legal move. Integrators
sometimes win, and that is architecture, not cowardice.

Coarse is cheaper until a disintegrator is existential
(security scope, a scale cliff, a release that cannot wait).
Fine is expensive until an integrator is existential (money
invariants, a workflow that is one button). Start as coarse
as the *current* disintegrators allow; do not start from
nanoservices and glue upward.

## Two kinds of fights (not the book’s plot)

The running ticketing/field-ops case stages two arguments
you will have under other names. Remember the **shapes**.

### Assignment-shaped: “is this *operation* a service?”

A core entity already has a service. Someone wants to peel
off a **verb** (assign, route, schedule, score) because:

- the verb’s code is volatile (new routing heuristics weekly),
- the verb has different scale (a solver vs CRUD),
- the verb has a different trust zone (only dispatchers),
- or the verb should be extensible (customer-specific rules).

Integrators fire back: the verb **writes the same rows** as
the entity service, needs a transaction with “create,” or is
step two of a three-step workflow the user thinks is one
click.

**Problem** — The operation is noisy, but it is not a
separate invariant.
**Solution** — Split the operation only if a disintegrator
is *measured* and you have a plan for the data (own a slice,
delegate writes, or accept eventual consistency).
**Failure mode** — `AssignmentService` that is a facade
over `UPDATE tickets SET assignee=…` plus a network hop.
You bought volatility isolation you did not use and paid
with a distributed write.

When this fight is real: the algorithm is a product, its
failure must not take down ticket CRUD, and its input can
be a message instead of a row lock. When it is fake: the
“algorithm” is a column update with extra logging.

### Registration-shaped: “is this *entity slice* a service?”

A user journey (register, onboard, open account) touches
profile, credentials, preferences, billing setup, maybe
notifications. Disintegrators say: credentials are a
security zone; billing is PCI; preferences churn.

Integrators say: the journey is **one workflow**, often
**one transaction** from the user’s point of view (“I signed
up” is true or it is not), and the data is born together.

**Problem** — You split onboarding into four services and
the user can exist in identity but not in billing, or the
other way around.
**Solution** — Keep a **coarse onboarding quantum** until a
specific slice (usually security or payments) pays for the
seam. After the account exists, later life-cycle operations
can live elsewhere.
**Failure mode** — Entity-per-noun (`ProfileService`,
`PreferenceService`) that must be orchestrated to create a
row the monolith inserted in one statement. You optimized
for an org chart, not for a journey.

Assignment-shaped fights are **verb vs entity**.
Registration-shaped fights are **journey vs noun**. Mixing
the two in one review is how you split both ways and get
neither benefit.

## How this shows up when you design something

- A proposal lists six disintegrators and no integrator.
  Ask what transaction, hop, or join you just denied.
- “Entity service” on the whiteboard: which fight is it?
  Verb or journey?
- Two services, one database: granularity theater
  ([ch. 6](../6-operational-data/) still applies).
- Merge proposal after a quarter of sagas: that is this
  chapter working as intended, not a failed microservices
  conversion.

## Check yourself

1. Name two disintegrators and two integrators for a
   service you have actually split (or refused to). What
   won, and what did you pay?
2. Why is “one class, one service” a failure mode of
   *scope*, not a reading of single responsibility?
3. Volatility vs throughput: both argue to split. How
   would the resulting seams differ? Give a production
   shape for each.
4. A bulkhead in one process vs a separate service: which
   failure does each *not* contain?
5. Security as a disintegrator: when is column encryption
   enough, and when do you need a process boundary?
6. “We’ll choreograph it” as a response to a transaction
   integrator: what user-visible failure did you just
   accept?
7. Shared *domain* code vs shared *platform* code: which
   one is a granularity integrator, and where does the
   other belong?
8. Draw an assignment-shaped fight in a domain you know
   (not ticketing). What would make the verb its own
   quantum, and what would make that a facade?
9. Draw a registration-shaped fight. Where does a
   coarse onboarding service beat four noun services?
10. Someone says the target is “about 25 services.” What
    in this chapter do you use to refuse a number and
    demand a balance sheet instead?

Continue to [Reuse patterns](../8-reuse-patterns/).

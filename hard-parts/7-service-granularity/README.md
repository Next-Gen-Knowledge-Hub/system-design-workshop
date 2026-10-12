# 7. Service granularity

These notes teach how big a service should be, from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## Size is a balance you re-run

Once data can live apart, the next question is how big a service should be. This is not a definition of “microservice,” and it is not a line-count rule. Granularity is a tug-of-war you re-run when a force changes.

An **architecture quantum** is the smallest piece you can deploy, scale, and restore on its own, including the data it writes. Two processes that share a writable schema are one quantum. A service boundary that does not include its data is granularity theater: you moved a process and left the bottleneck in the shared tables.

Forces that argue for smaller or at least separate services are **disintegrators**. Forces that argue for keeping work together, or for merging a split you regret, are **integrators**. Granularity is the net of those forces, not a target number of services. None of the disintegrators wins alone.

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

Both ends fail. Split on the slogan “single responsibility” and every user request becomes a five-hop workflow. Refuse to split and a harmless copy change redeploys a payments process. Those are the same kind of mistake at opposite ends.

Two services with the same entity name can both be right at different times. Load, a regulation, or a release rate moved. Copying a ticketing split because a book’s field-ops case split something similar is cargo cult. Steal the kind of fight, not the plot.

“We need a transaction” is an integrator in this lesson: should these be two processes? Isolation anomalies inside one shared database — what a concurrent transaction can still observe — are a different question, answered by the isolation level of that one engine. Stay here for the process boundary.

## Disintegrators

### Service scope and function

One process that does two jobs a user would name differently — “catalog browse” and “settle payment” — makes every change description a lie. You risked billing to ship a badge color.

Draw the service around a purpose you can say in one breath, and around a contract other teams can depend on. A **contract** is the versioned shape of requests and events those teams are allowed to rely on. Purpose is not one endpoint, and it is not one entity. A payments service may own authorize, capture, and refund: one purpose, several operations.

“Single responsibility” read as “one function per repository” builds a distributed function-call graph. Scope was supposed to *reduce* the reasons to change, not to maximize Git repositories. If you cannot explain why the service exists without listing three unrelated nouns, it is too coarse, or you have not found its purpose yet. Those are different bugs. The first needs a split. The second needs a clearer purpose before any split.

### Code volatility

A volatile corner — pricing rules, integration adapters, an experimental backend-for-frontend — lives in the same deployable as a stable core: ledger posting, an entitlement check. You ship the core as often as the experiment. Each deploy is a chance to break what was not changing. Deploys are not the villain. Coupling the release trains is.

Split along the rate of change. Stable code should not ride the release train of fidgety code.

Splitting by file type — “all utilities,” “all controllers” — still redeploys the ledger when a data-transfer object moves. Volatility is a time series, not a folder. A **fitness function** — an automated check that fails when a structural decision has gone stale — can measure commit density and incident density per component. The candidates to extract are the outliers, not the averages.

### Scalability and throughput

One endpoint is 90% of CPU. The rest is idle admin. You scale the whole process, including the parts that do not need replicas, or you cannot scale the hot path without cloning a giant heap.

Split so the hot operation is its own deployable, with its own capacity, its own cache, and its own service-level objective. Throughput islands deserve isolation even when the code looks related.

Splitting a cold path “for symmetry” pays the distributed tax for a function that ran twice an hour. A scale argument needs a graph, not a feeling.

This is *service* scale: processes, heaps, replicas. Database scale is a separate cut — hot and cold tables sharing a buffer pool and a write-ahead log. If the hot path still shares a write schema with the cold path, you moved the bottleneck one layer down. The processes scaled. The quantum did not.

### Fault tolerance

A flaky dependency or a leak in a non-critical feature takes down the process that also serves the critical path. One out-of-memory kill, one exhausted thread pool, one poison message: shared fate.

Separate the failure domains. The thing that may collapse — image resize, an outbound fax, a ranking model — should not share a runtime with “place order.”

A bulkhead inside one process (a separate thread pool, a separate queue) contains some resource exhaustion. It does not contain a bad deploy. Both sides still ship in the same binary. Bulkheads are good. They are not an independent quantum, and they are a poor permanent stand-in for a split you already know you need.

If a library wedges the event loop, ask what else dies. When the answer includes money or login, a disintegrator and an integrator are fighting. Name both before you cut.

### Security

Secrets, personal data, or card data share a process, a container image, and an identity role with a public feature. The attack surface is the union. Auditors treat the union as in scope.

Split so the trust zone is small: fewer endpoints, a tighter network policy, separate key material, a separate audit log. Card-data handling in the same box as a marketing content system is how audit scope explodes.

“We encrypt the column” while the web tier still has `SELECT` on the vault is not a granularity boundary. Encryption hides bytes. A service boundary with no route from the public site is the boundary.

Security splits are expensive: two on-call rotations, two identity stories. Use them when the threat model or the regulator pays you back. Do not use them to look serious.

### Extensibility

Every new customer variant is an `if` in the core service. You cannot add a plugin without shipping the engine. The roadmap becomes a permutation matrix.

Extract the variation point: a rules service, an extension process beside the core, a webhook owner — something you can deploy without republishing the stable core.

A “plugin service” that is a scripting engine still running inside the core, with extra network steps, does not extract the variation. A zoo of one-off services with no shared contract does not either. Extensibility without a contract is more code in more places. You feel this disintegrator late: the service works, and then the seventh variant makes the eighth unshippable.

## Integrators

### Database transactions

Two operations must commit together or not at all: reserve inventory and open the ticket, or take payment and record the entitlement. In one service and one database, that is a transaction. In two services, it is a protocol.

If the invariant is synchronous and local, keep one service, and likely one data domain. If you split anyway, you are volunteering for an ownership rule and for a saga, not for “a smaller class.” **Ownership** means exactly one service holds the write credentials for a table. A **saga** is a multi-step action in which each owner commits locally and a later failure runs a compensating action instead of one `ROLLBACK`.

**Two-phase commit** across the two databases, used as the happy path, asks each database to prepare and then to commit or abort together. As an architecture it couples their availability: neither side can finish while the other is in doubt. You lost the granularity and kept the coordination tax. Atomic commit of the pair is also not the isolation a single engine would have given you across both tables. You got “both committed or neither did,” which is a narrower gift.

How strong this integrator is depends on how bad a partial success is. “The email did not send” is not “the money moved twice.”

### Workflow and choreography

A single user action — register, check out, assign — becomes a conversation among many services: eight calls, three timeouts, and a compensating path nobody can draw. **Choreography** means each service reacts to the previous step and nobody holds the whole picture. An **orchestrator** is a process that holds the in-flight state and calls each owner in order.

If the workflow *is* the product, consider one service for the duration of that action, or accept an explicit orchestrator whose job is that state. Chatty choreography is a smell that granularity went past the user step.

“We will make it async” hides the hop count from the request path and moves it into “why is the user interface still spinning.” Asynchronous messaging is a tool. It does not pardon a workflow that was split past the button the user pressed.

Count synchronous hops on the critical path. Each hop is a service-level objective you inherited. If you cannot name the hop budget, the design is not finished.

### Shared code

Two would-be services share a thick domain module: pricing, tax, entitlement. Split them and you either copy the module, extract a library that couples their deploys (every consumer must rebuild to pick up a change), or extract a shared service that becomes the new monolith. Those three, plus a node-local sidecar for plumbing, are reuse choices. They are how you share a behavior once you have decided it should be shared. They are not, by themselves, a reason the two domains are one service.

If the shared code *is* the domain, you may have one service. If it is infrastructure — logging, authentication middleware — it is not a granularity argument. Take it to the reuse menu. Do not confuse the two.

`company-commons` as the reason two unrelated domains cannot release is the failure. Shared *domain* code is an integrator: it pulls the services back together. Shared *platform* code is a reuse choice: a library or a sidecar, kept free of domain types.

### Data relationships

The two services would always join. The tables still want to live together. Collocate the services, or collocate the data and accept that they are one quantum. Relationships that are write-time invariants are stronger integrators than read-time decorations. A decoration can be a later read pattern: call the owner at read time, copy a few columns into the reader’s store, cache hot keys, or stand up a serving view whose product is the combination.

An “entity service” per table — `Customer`, `Address`, `Email` — maximizes relationship pain. A table is not a service. That is the classic overshoot: you split nouns the engine used to join, and every screen becomes a network conversation.

## Finding the balance

No formula outputs a service count. The argument you can repeat is:

1. List the disintegrators that hurt this quarter, not the ones that are true in theory.
2. List the integrators that would fire if you split.
3. Estimate the cost of the seam: latency, who writes, on-call, and a saga if the invariant crosses owners.
4. Split or merge **one** seam. Measure. Repeat.

A once-and-for-all map of services for the next five years will not survive the next regulation or the next hot endpoint. Treat the granularity decision as a record you expect to revise: context, choice, consequence. Fitness functions that keep it honest are hop count on a user journey, deploy frequency per service, and whether two services share an error budget.

A target slide with forty boxes, none of which match yesterday’s incidents, is a failure. Refusing to merge is another. Merge is a legal move. When integrators win, that is architecture.

Coarse is cheaper until a disintegrator is existential: security scope, a scale cliff, a release that cannot wait. Fine is expensive until an integrator is existential: a money invariant, a workflow that is one button. Start as coarse as the *current* disintegrators allow. Do not start from nanoservices and glue upward.

A proposal that lists six disintegrators and no integrator has not named the transaction, the hop, or the join it just denied. Two services and one database are granularity theater: the writable schema is still one quantum. A merge proposal after a quarter of sagas is this balance working, not a failed conversion to small services. “About twenty-five services” is a number. The balance sheet above is the design.

## Two shapes of fight

The ticketing and field-ops case stages two arguments you will have under other names. Remember the shapes.

### Assignment-shaped: is this operation a service?

A core entity already has a service. Someone wants to peel off a verb — assign, route, schedule, score — because the verb’s code is volatile (new routing heuristics every week), because the verb has different scale (a solver versus ordinary create-read-update), because the verb has a different trust zone (only dispatchers), or because the verb should be extensible (customer-specific rules).

The integrators answer: the verb writes the same rows as the entity service, it needs a transaction with “create,” or it is step two of a three-step workflow the user thinks is one click.

The operation can be noisy without being a separate invariant. Split it only when a disintegrator is measured *and* you have a plan for the data: the verb owns a slice of the rows, it delegates the write to the entity service (the entity service remains the only writer and the verb asks it to mutate), or you accept that the two facts converge after each has committed locally.

An `AssignmentService` that is a facade over `UPDATE tickets SET assignee = …` plus a network hop bought volatility isolation it did not use and paid with a distributed write.

The fight is real when the algorithm is a product, its failure must not take down ticket create-read-update, and its input can be a message instead of a row lock. The fight is fake when the “algorithm” is a column update with extra logging.

### Registration-shaped: is this entity slice a service?

A user journey — register, onboard, open an account — touches profile, credentials, preferences, billing setup, and maybe notifications. Disintegrators say credentials are a security zone, billing is card data, and preferences churn. Integrators say the journey is one workflow, often one transaction from the user’s point of view (“I signed up” is true or it is not), and the data is born together.

Split onboarding into four services and the user can exist in identity and not in billing, or the other way around. Keep a coarse onboarding quantum until a specific slice, usually security or payments, pays for the seam. After the account exists, later life-cycle operations can live elsewhere.

`ProfileService` and `PreferenceService`, orchestrated to create a row the monolith inserted in one statement, optimize for an org chart. The journey was the unit.

Assignment-shaped fights are verb versus entity. Registration-shaped fights are journey versus noun. Mixing the two in one review is how you split both ways and get neither benefit. On a whiteboard, an “entity service” has to say which fight it is.

## Recap

Granularity is disintegrators minus integrators, re-measured when load, regulation, or release rate changes. Scope, volatility, throughput, fault isolation, security zones, and extension points pull services apart. Local transactions, chatty workflows, thick shared domain code, and write-time joins pull them together. Split or merge one seam, then measure hops, deploy frequency, and shared error budgets. An assignment fight peels off a verb only when the data plan is real. A registration fight keeps the journey coarse until security or payments pays for the seam. A target count of services is not a design.

# 8. Reuse patterns

These notes teach the reuse menu from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## Sharing is a coupling decision

Once work sits in separate quanta, the next temptation is to share the bits that look the same. An **architecture quantum** is the smallest piece you can deploy and restore on its own, including the data it writes. Reuse is not a virtue. It is a coupling decision with a menu: copy the code, ship a library, stand up a service, or run a sidecar. Each item fails differently.

```
  what you are sharing          how it actually couples

  source snippet                nothing at runtime
       |                        drift over time
       v
  shared library                compile / build / version
       |                        every consumer rebuilds
       v
  shared service                network + availability
       |                        one objective for every caller
       v
  sidecar / mesh                node-local process
                                infrastructure reuse, not domain reuse

  domain logic                  infrastructure logic
  (pricing, tax,                (mutual TLS, retries, metrics,
   “how we refund”)              authentication plumbing)
```

Reuse buys a single place to change. It sells a constraint on how that change rolls out. “Do not repeat yourself” is a hint to look at this menu. It is not an architecture argument.

Sharing domain brains and sharing infrastructure plumbing look alike on a slide and are opposite operational bets. Mixing them is how a logging library grows a `CustomerUtil`. Paste a tax calculator into nine repositories and you will patch eight of them. Stand up a `CommonService` that sits on every critical path and is owned by nobody, and you intended only one of those outcomes. Both are reuse.

## Code replication

Copy the code, or generate it and check it in. There is no shared artifact and no shared process.

Several quanta need the same tiny behavior — a header format, a checksum, a mapping table of ten rows — and you do not want a release train. Replicate. Own a copy in each repository. When the behavior is stable and small, drift is cheaper than coordination.

Replication is the default for domain code you are not sure should be shared. It feels sloppy. It keeps quanta independent. A later library is an upgrade you can justify with a CVE — a published security defect — or a weekly bug. Starting with a library is a downgrade you cannot undo without a versioning saga: every consumer has to move versions before the coupling loosens.

What breaks is replicating volatile, correctness-sensitive logic: tax, cryptography, authorization. You will miss a patch. The incident report will say “we thought that repository had the fix.”

When you replicate, write down what would make you stop: a second CVE, a third consumer, a regulator. Otherwise copies proliferate with no owner for the idea.

## Shared library

One artifact, many builds. Consumers compile or bundle it. A change propagates when they upgrade and redeploy.

### Dependencies

The library pulls a graph: logging, metrics, an HTTP client, a particular JSON library or collections package such as Jackson or Guava. Two services cannot upgrade because `commons` pinned a transitive dependency full of known vulnerabilities.

Keep domain libraries thin: little or no framework. Infrastructure belongs in the platform, not in `billing-core`. Treat dependency edges as part of the public contract. A fitness check you can automate is the shape of that graph: a thin domain library stays free of a web framework and of other domains’ types; an uber-jar that “just adds tracing” fails the check.

A library with an opinionated web framework inside is not a library. It is a product that hijacked your process. `company-commons-2.14` that “just adds tracing” and now every service shares a runtime is a distributed monolith at **build time**.

### Versioning

You cannot change a signature without breaking someone who cannot deploy this week. Use explicit versions, a compatibility window, and a rule for how many major versions may run in production at once. A breaking change needs a migration path, not an announcement in chat.

Everyone on a snapshot of the main branch is a secretly shared codebase. Never upgrading is replication with extra steps and a false sense that one fix covers the fleet.

Semantic versioning helps if you honor it. It does not help if a “minor” release includes a behavioral change in rounding. Versioning is a social contract backed by tests, not a number in Gradle.

### Change risk

A bug fix in a library is not deployed until every consumer ships. That is the deal. You bought consistent *source*. You did not buy consistent *runtime*.

A security fix that must land everywhere by Friday needs a platform pipeline that bumps and rolls consumers. If you do not have that pipeline, you picked the wrong pattern: you needed a sidecar or a service you can patch once.

“It’s in the library, we’re safe” is false until you can list the versions running in production. If you cannot list them, you replicated and forgot.

Libraries shine for stable, in-process, pure functions: encoders, policy parsers, identifier formats, math. They sour for anything that must change faster than the slowest consumer.

On-call for `platform-commons` means that library has become a runtime. Admit it, and give it the operations of a runtime, or shrink it back to pure functions.

## Shared service

One runtime, many callers. A change propagates when *you* deploy. Callers pay a network hop and an availability dependency.

Many quanta need the same behavior at the same version at the same time: exchange rates, a central authorization decision, a hardware-backed signing key. Give that a service with an explicit contract, a service-level objective, and an owner. Version the API. The owner is on call for the callers’ outage, because the outage will be theirs.

Extracting a service to “reuse a class” turns a function call into a distributed system. If the logic is not a product with a life cycle, keep a library or a copy. “Let’s make a shared service for this enum” is the same mistake. Enums are not products. Replicate them, or put a stable encoding in a library.

### Change risk

One deploy can break every consumer at once. Prefer additive API changes, parallel versions, and checks driven by what consumers actually send. The owner’s release process *is* the company’s release process for that seam.

Breaking a JSON field on a Friday because “nobody uses that field” is stamp coupling: callers depend on the exact shape of your payload, so a rename or a dropped field breaks the fleet. A shared service concentrates political change risk, not only technical risk. Someone has to be allowed to say no to the Friday change.

Against a library, the trade is this. Libraries stagger breakage: good for blast radius, bad when the value must be identical today. Services synchronize breakage: good for consistency, bad for Friday.

A CVE in rounding, when every product must round the same way on the same day, is a service, or a forced library bump with an inventory of who has shipped. A CVE in TLS, when the fix would need twenty domain deploys, wanted a sidecar or a platform image, not a copied stack.

### Performance

A hop on the critical path is latency you cannot optimize inside the caller: serialization, TLS, retries, and the tail latency of the shared heap.

A reused decision that sits in every request — authorize, price, a feature flag — has to be measured. Cache it with a documented staleness. That cache is already a read pattern: the caller holds a copy and must say how wrong the copy is allowed to be. Or keep the logic in-process, as a library or a copy, when microseconds matter and identical versions do not.

N round trips to `CommonService` per page make reuse the throughput ceiling.

### Scalability

The shared service is a hotspot. Its capacity plan is the sum of every consumer’s worst day. A viral feature in one product knocks out pricing for all products.

Quotas, isolation, and independent scale are the minimum. The shared service needs better operations than its callers, not worse. If you cannot staff that, do not share a runtime.

One cluster, no story for a noisy neighbor, and a single database underneath that every caller’s facts still join through, rebuilds the monolith at the reuse layer. The services look separate. The hotspot and the shared writable schema are the system.

### Fault tolerance

If the shared service is down, every consumer feature that needs it is down. A circuit breaker degrades the call. It does not invent the tax amount.

Login, checkout, and search that all import the same dependency share a fate. Either the shared service has a stricter objective than any caller, or each caller has a defined degraded mode: fail closed, or fail open. Pick one and write it down.

Retry storms finish the outage. The reuse pattern chose a common fate. The retry policy must not amplify it.

Shared services are justified when identical runtime behavior is the requirement: a single ledger of rates, a single place keys live. They are not justified by a diagram with a `Util` box.

## Sidecars and a service mesh

A **sidecar** is a process next to yours on the same node: a proxy, an agent, a daemon. A **service mesh** is that idea as a platform: data-plane sidecars plus a control plane that pushes policy.

Every service reimplements mutual TLS, retries, timeouts, a standard metric, or inbound authentication. That reuse is infrastructure. A shared *domain* service would sit on the wrong path and crash the wrong way: a pricing bug would take down the proxy path that every request uses.

Push cross-cutting plumbing into a sidecar or a mesh so domain binaries stay thin, and so you can patch the plumbing once per node or image without a domain release.

A mesh as a personality — custom filters that encode business rules such as pricing or entitlement — hides a shared domain service in Envoy, the proxy most meshes use as their data plane. Nobody knows where the rule lives, and the change risk is still there, only less visible. An honest shared service at least has an owner and a contract. A rule buried in a filter has neither.

Sidecars cost CPU, latency, and cognitive load. They pay for themselves when the alternative is twenty slightly wrong retry implementations. They do not pay for themselves as a place to stash domain reuse you were afraid to write down as a decision.

For retries, a library still requires a rebuild to change policy. A mesh can change timeouts from the control plane. That is the point, and the danger: central policy, local outages. One bad push hits every node.

## When reuse is worth the coupling

Reuse is worth the coupling when most of these are true:

1. There are many consumers, not “maybe two next year.”
2. Correctness is shared. Getting it wrong twice is an incident, not a style issue.
3. The change rate fits the pattern. Fast, and it must be identical: a service or a sidecar. Slow, and it runs in-process: a library. Slow, and each quantum may drift: a copy.
4. Someone owns the artifact, with an objective or a version promise. “The platform team might look at it” is not an owner.

A design review that says “put it in shared so we don’t repeat ourselves” has not picked a menu item. Walk the menu out loud. Pick one. Write the failure you accept into the decision record.

Reuse as morality — “do not repeat yourself” applied to two lines of data-transfer mapping — couples deploys to save a snippet.

Ask the inverse. What if we never unify these? If the answer is mild drift, copy. If the answer is “we overcharge a customer in one channel,” do not copy.

A shared library that blocks two teams’ deploys is a versioning problem, a replication problem, or a sign you needed a service. Versioning is the next move when the artifact is stable and pure and the teams can carry more than one major. Replication is the next move when the behavior may drift and correctness is local. A service is the next move when the value must be identical at runtime this hour. Each of those is the wrong move when the condition is missing: a service for an enum, a forced single version for code that was allowed to drift, a copy of a tax rule that must match today.

## Reuse through a platform

An internal platform is paved roads: identity, deploy, telemetry, secrets, maybe a mesh. That is reuse of an operating model, not a `UserService` for the whole company.

Each team’s private “reuse” is often a snowflake library that implements the same three concerns differently. Platform teams provide defaults with escape hatches. Domain teams consume the roads. They do not negotiate a shared domain model with the platform.

A platform that owns `Order` because “everyone has orders” rebuilds the monolith in the platform organization. Platform reuse is infrastructure logic, plus documentation and golden paths.

Success looks like a boring sidecar, a blessed library *without* domain types, and a pipeline. It does not look like a canonical data model. Who may write the customer row or the order row is an ownership question: one service holds the write credentials. A platform diagram does not answer it.

## Domain functionality and infrastructure logic

These are different decisions. Use different patterns.

| | Domain functionality | Infrastructure logic |
|---|---|---|
| Examples | Price a job, refund policy, “what is a customer id” | Mutual TLS, retries, log shape, process identity |
| Copy | Reasonable if rules may drift by product | Dangerous for security plumbing |
| Library | Thin, stable, pure; versioned | Reasonable if it has no domain types |
| Shared service | Only if it is a product with an objective | Rarely; it becomes a bottleneck on every call |
| Sidecar or mesh | Almost never | The intended home |

A “shared kernel” that mixes `TaxCalculator` with `HttpClientFactory` has to be split. Domain reuse needs an owner on a domain team. Infrastructure reuse needs a platform owner. If you cannot name which, you are not ready to share.

`commons` that every service waits on, and that waits on every domain’s types, is the highest-coupled quantum in the estate. You undid the granularity split: unrelated domains cannot release because the shared kernel moved.

A practical test: would a new product line be allowed to diverge? If yes, the behavior was never a candidate for a shared domain service. If no — a statute, cryptography, a brand-wide pricing law — you still choose library versus service by rollout speed and hop cost. The word “must” does not pick the pattern.

“Retry with jitter” is infrastructure: a sidecar or a mesh, or a thin library with no domain types. “Who is allowed to refund” is domain: a copy if product lines may differ, a thin library if the rule is stable and in-process, a service only when every channel must apply the same rule at the same time and someone owns that objective.

## Recap

Copy, library, service, and sidecar each buy one place to change and sell a different rollout constraint. Copy when the behavior is small, stable, or allowed to drift; stop copying when a vulnerability, a third consumer, or a regulator appears. A library gives consistent source after each consumer redeploys, and it stays thin. A service gives identical runtime behavior and concentrates latency, scale, and outage on one owner. A sidecar or mesh patches infrastructure once per node and does not hold pricing rules. Domain logic and plumbing do not share a kernel. Reuse is worth it when many consumers, shared correctness, a matching change rate, and a real owner are all true.

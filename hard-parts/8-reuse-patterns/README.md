# 8. Reuse patterns

Companion notes for **Chapter 8** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021). Chapters 6–7 pulled
work into separate quanta. The next temptation is to **share** the bits
that look the same. Reuse is not a virtue. It is a coupling decision
with a menu: copy, library, service, sidecar. Each menu item fails
differently.

Skip it and you will either paste a tax calculator into nine repos
(and patch eight of them) or you will stand up a `CommonService`
that is on every critical path and owned by nobody. Both are reuse.
Only one was intended.

## The mental model

```
  what you are sharing          how it actually couples

  source snippet                nothing at runtime
       |                        drift over time
       v
  shared library                compile / build / version
       |                        every consumer rebuilds
       v
  shared service                network + availability
       |                        one SLO to rule them all
       v
  sidecar / mesh                node-local process
                                infra reuse, not domain reuse

  domain logic  ──?──  infrastructure logic
  (pricing, tax,        (mTLS, retries, metrics,
   “how we refund”)      authn plumbing)
        ^                      ^
        different decisions ----+
```

The sentence to keep: **reuse always buys a single place to change,
and always sells a constraint on how that change rolls out.**

Two consequences. First, “DRY” is not an architecture argument; it
is a hint to *look* at this menu. Second, sharing **domain** brains
and sharing **infrastructure** plumbing look similar in a slide and
are opposite operational bets. Mixing them is how a logging library
grows a `CustomerUtil`.

## Code replication

Copy the code (or generate it and check it in). No shared artifact.
No shared process.

**Problem** — Several quanta need the same tiny behavior (a header
format, a checksum, a mapping table of ten rows) and you do not
want a release train.
**Solution** — Replicate. Own a copy in each repo. If the behavior
is stable and small, drift is cheaper than coordination.
**Failure mode** — Replicating **volatile, correctness-sensitive**
logic (tax, crypto, authorization). You will miss a patch. The
incident report will say “we thought that repo had the fix.”

Replication is the **default for domain code you are not sure
should be shared**. It feels sloppy. It keeps quanta independent.
A later library is an upgrade you can justify with a CVE or a
weekly bug; starting with a library is a downgrade you cannot
undo without a versioning saga.

When you replicate, write down **what would make you stop**: a
second CVE, a third consumer, a regulator. Otherwise copies
proliferate with no owner for the *idea*.

## Shared library

One artifact, many builds. Consumers compile or bundle it. Change
propagates when they **upgrade and redeploy**.

### Dependencies

**Problem** — The library pulls a graph (logging, metrics, an HTTP
client, a particular Jackson/Guava). Two services cannot upgrade
because `commons` pinned a CVE-ridden transitive.
**Solution** — Keep domain libraries **thin**: little or no
framework. Infrastructure belongs in the platform, not in
`billing-core`. Treat dependency edges as part of the public
contract.
**Failure mode** — `company-commons-2.14` that “just adds
tracing” and now every service shares a runtime. You built a
distributed monolith at **build time**.

A library with an opinionated web framework inside is not a
library. It is a product that hijacked your process.

### Versioning

**Problem** — You cannot change a signature without breaking
someone who cannot deploy this week.
**Solution** — Explicit versions, a compatibility window, and a
rule for **how many majors** may run in production. Breaking
changes need a migration path, not a Slack announcement.
**Failure mode** — Everyone on a snapshot of main, or never
upgrading. The first is a secretly shared codebase; the second
is replication with extra steps and a false sense of a single
fix.

Semver helps if you honor it. It does not help if “minor”
includes a behavioral change in rounding. Versioning is a
social contract backed by tests, not a number in Gradle.

### Change risk (library)

A bug fix in a library is **not** deployed until every consumer
ships. That is the deal. You bought consistent *source*; you did
not buy consistent *runtime*.

**Problem** — A security fix must land everywhere by Friday.
**Solution** — Either you have a platform pipeline that bumps
and rolls consumers, or you picked the wrong reuse pattern and
needed a sidecar/service you can patch once.
**Failure mode** — “It’s in the library, we’re safe.” Inventory
of versions in prod is the real safety. If you cannot list them,
you replicated and forgot.

Libraries shine for **stable, in-process, pure** functions:
encoders, policy parsers, ID formats, math. They sour for
anything that must change **faster than the slowest consumer**.

## Shared service

One runtime, many callers. Change propagates when **you** deploy.
Callers pay a network hop and an availability dependency.

**Problem** — Many quanta need the same behavior **at the same
version at the same time** (exchange rates, a central
authorization decision, a hardware-backed signing key).
**Solution** — A service with an explicit contract, SLO, and
owner. Version the API. Make the owner on-call for *your*
outage, because it will be.
**Failure mode** — Extracting a service to “reuse a class.”
You turned a function call into a distributed system. If the
logic is not a product with a life cycle, keep a library or a
copy.

### Change risk (service)

**Problem** — One deploy can break every consumer at once.
**Solution** — Additive API changes, parallel versions, and
consumer-driven checks. The owner’s release process *is* the
company’s release process for that seam.
**Failure mode** — Breaking JSON on a Friday because “nobody
uses that field.” Stamp coupling belongs in
[ch. 13](../13-contracts/); here, remember that a shared
service concentrates **political** change risk, not just
technical.

The trade vs library: libraries stagger breakage (good for
blast radius, bad for “must be identical today”). Services
synchronize breakage (good for consistency, bad for Friday).

### Performance

A hop on the critical path is latency you cannot optimize in
the caller. Serialization, TLS, retries, tail latency of the
shared heap.

**Problem** — A reused decision sits in every request
(authorize, price, feature flag).
**Solution** — Measure. Cache with a documented staleness
(that cache is already [ch. 10](../10-distributed-data-access/)
thinking). Or keep the logic in-process (library/copy) if
microseconds matter and consistency of version does not.
**Failure mode** — N round-trips to `CommonService` per page.
Reuse became the throughput ceiling.

### Scalability

The shared service is a **hotspot**. Its capacity planning is
the sum of every consumer’s worst day.

**Problem** — A viral feature in one product knocks out
pricing for all products.
**Solution** — Quotas, isolation, independent scale — the
shared service needs *better* ops than its callers, not worse.
If you cannot staff that, do not share a runtime.
**Failure mode** — One cluster, no noisy-neighbor story, a
single database from [ch. 6](../6-operational-data/) under it.
You re-monolithed at the reuse layer.

### Fault tolerance

If the shared service is down, **every consumer’s feature that
needs it is down**. Circuit breakers degrade; they do not
invent the tax amount.

**Problem** — Login, checkout, and search all import the same
dependency.
**Solution** — Either the shared service has a stricter SLO
than any caller, or callers have a **defined degraded mode**
(fail closed vs fail open — pick, write it down).
**Failure mode** — Retry storms that finish the outage. The
reuse pattern chose a common fate; your retry policy must
not amplify it.

Shared services are justified when **identical runtime
behavior** is the requirement (a single ledger of rates, a
single place keys live). They are not justified by a UML
diagram with a `Util` box.

## Sidecars and service mesh

A **sidecar** is a process next to yours: proxy, agent,
daemon. A **service mesh** is that idea as a platform: data
plane sidecars plus a control plane that pushes policy.

**Problem** — Every service reimplements mTLS, retries,
timeouts, gold-metric emit, or inbound authn. That reuse is
*infrastructure*, but a shared *domain* service would sit on
the wrong path (and crash the wrong way).
**Solution** — Push cross-cutting **plumbing** into a sidecar
or mesh so domain binaries stay thin and you can **patch the
plumbing once per node/image** without a domain release.
**Failure mode** — A mesh as a personality: custom filters
that encode *business* rules (pricing, entitlement). You hid
a shared domain service in Envoy. Now nobody knows where the
rule lives, and you still have change risk — only less
visible.

Sidecars cost **CPU, latency, and cognitive load**. They pay
for themselves when the alternative is twenty slightly wrong
retry implementations. They do not pay for themselves as a
place to stash domain reuse you were afraid to ADR.

Mesh vs shared library for retries: the library still
requires a rebuild to change policy. The mesh can change
timeouts centrally. That is the point — and the danger
(central policy, local outages).

## When reuse actually adds value

Reuse is worth the coupling when most of these are true:

1. **Many consumers** (not “maybe two next year”).
2. **Correctness is shared** (getting it wrong twice is an
   incident, not a style issue).
3. **The change rate fits the pattern**: fast + must be
   identical → service or sidecar; slow + in-process →
   library; slow + independent → copy.
4. **Someone owns the artifact** with an SLO or a version
   SLA. “The platform team might look at it” is not an owner.

**Problem** — A design review says “put it in shared so we
don’t repeat ourselves.”
**Solution** — Walk the menu above out loud. Pick one. Write
the failure mode in the ADR.
**Failure mode** — Reuse as morality. DRY applied to two
lines of DTO mapping. You coupled deploys to save a gist.

Also ask the inverse: **what if we never unify these?** If
the answer is “mild drift,” copy. If the answer is “we
overcharge a customer in one channel,” do not copy.

## Reuse via platforms

An internal platform is **paved roads**: identity, deploy,
telemetry, secrets, maybe a mesh. That is reuse of
*operating model*, not a `UserService` for the whole
company.

**Problem** — Each team’s “reuse” is a snowflake library
that implements the same three concerns differently.
**Solution** — Platform teams provide **defaults with
escape hatches**. Domain teams consume roads; they do not
negotiate a shared domain model with the platform.
**Failure mode** — Platform that owns `Order` because
“everyone has orders.” You rebuilt the monolith in the
platform org. Platform reuse is **infrastructure logic**
(next section), plus documentation and golden paths.

Platform success looks like a boring sidecar, a blessed
library *without domain types*, and a pipeline. It does not
look like a canonical data model. Canonical data is
[ownership](../9-data-ownership/), not a platform win.

## Shared domain functionality vs shared infrastructure logic

These are different decisions. Use different patterns.

| | Domain functionality | Infrastructure logic |
|---|---|---|
| Examples | Price a job, refund policy, “what is a customer id” | mTLS, retries, log shape, process identity |
| Copy | OK if rules may drift by product | Dangerous for security plumbing |
| Library | Thin, stable, pure; versioned | OK if it has no domain types |
| Shared service | Only if it is a product with SLO | Rarely; it becomes a bottleneck on every call |
| Sidecar / mesh | Almost never | The intended home |

**Problem** — A “shared kernel” mixes `TaxCalculator` with
`HttpClientFactory`.
**Solution** — Split the kernel. Domain reuse needs an
owner in a *domain* team. Infra reuse needs a *platform*
owner. If you cannot name which, you are not ready to share.
**Failure mode** — `commons` becomes the highest-coupled
quantum in the estate: every service waits on it, and it
waits on every domain’s types. That is the opposite of
[granularity](../7-service-granularity/).

A practical test: **would a new product line be allowed to
diverge?** If yes, it was never a candidate for a shared
domain service. If no (legal, crypto, brand-wide pricing
law), you still choose library vs service by **rollout
speed and hop cost**, not by the word “must.”

## How this shows up when you design something

- “Let’s make a shared service for this enum.” Replication
  or a library. Enums are not products.
- A CVE in TLS: if the fix needs twenty domain deploys, you
  wanted a sidecar/platform, not a copied stack.
- A CVE in rounding: if products *must* round the same day,
  that is a service or a forced library bump with inventory.
- On-call for `platform-commons`: that library has become a
  runtime. Admit it or shrink it.

## Check yourself

1. Pick a piece of code you have seen copied. Which of the
   four patterns should it have used, and what failure mode
   did the actual choice hit?
2. Why does a library not give you “one place to patch in
   production”? What extra machine do you need?
3. Dependency graph of a “thin” domain library vs a commons
   uberjar: what fitness function would you automate?
4. Shared service vs library for a value that must be
   *identical this hour* vs *identical this quarter*. Which
   pattern for which, and why?
5. Name a performance failure mode of a shared service that
   a sidecar would not have (or vice versa).
6. When does a service mesh hide a domain rule, and why is
   that worse than an honest shared service?
7. List the four “reuse adds value” tests. Kill a fake reuse
   proposal with one of them.
8. Platform paved road vs canonical `Customer` service: which
   is infrastructure reuse, and what chapter owns the other?
9. Domain vs infra: put “retry with jitter” and “who is
   allowed to refund” on the table. Pattern for each.
10. Merge pressure: a shared library is blocking two teams’
    deploys. Is the next move versioning, replication, or a
    service — and what would make that the wrong move?

Continue to [Data ownership](../9-data-ownership/).

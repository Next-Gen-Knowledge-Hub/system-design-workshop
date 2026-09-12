# 5. Component-based decomposition patterns

Companion notes for **Chapter 5** of *Software Architecture: The Hard Parts*
(Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani; O'Reilly,
October 2021).

Chapter 4 said you may choose component-based decomposition when the
graph is visible enough to work. This chapter is **the sequence of
moves** — not a pile of optional tricks. Skip it (or shuffle it) and
you will extract the wrong grain, hide coupling in nested packages,
and call a random cluster of classes a "domain service." The patterns
exist to make a seam you could put in an ADR and a fitness function.
They are still *code* moves; they do not split the database. That is
[ch. 6](../6-operational-data/).

## The mental model

```
  1 identify + size          "what is a component here?"
           |
           v
  2 gather common domain     shared nouns in one place, on purpose
           |
           v
  3 flatten                  nested trees stop lying
           |
           v
  4 dependencies             arrows, cycles, Ca/Ce at that grain
           |
           v
  5 component domains        clusters that might become quanta
           |
           v
  6 domain services          a deployable, if the driver still holds
```

The one sentence to remember a year from now: **decomposition is a
pipeline of visibility** — you size, gather, flatten, and map before
you are allowed to mint a service.

Two consequences fall straight out of that diagram. First, jumping to
step 6 because Kubernetes is ready produces a distributed monolith
with extra YAML. Second, stopping at step 4 is still a win: you may
have bought maintainability and testability inside one quantum
([ch. 3](../3-modularity/)) without paying for a second database.

For each pattern below: the problem, what you do, a fitness function
you could actually automate, and the failure if you skip it. Use the
book's ticketing/field-ops case as a *label* for examples, not as a
plot.

## Identify and size components

**Problem** — The unit of thought is wrong. Classes are too small to
be architecture; "the application" is too big; folders are layers
(`api/`, `domain/`, `infra/`) that cut across every change-reason.
Nobody can answer "what would we extract?" without waving at a
screen.

**What you do** — Name **components** at the grain a domain expert
could point to: intake, assignment, dispatch, invoicing, inventory,
notifications — not `InvoiceController` and not `helpers`. A
component is a clump of code that changes for one reason and that
you might one day deploy apart. Size it so that:

- Too small (a single class, a single function) will become a nano
  "service" that still chatters to everyone.
- Too large (half the monolith) will not move and cannot be tested
  as a piece.

Practical tells: a package (or module, or project) whose name is a
**noun-verb the business uses**, a handful of related types, and a
test target you can run without booting the world. If you cannot
draw twenty-or-fewer boxes, you have not identified components; you
have listed files.

Do not confuse this with [ch. 2](../2-coupling/) quanta. A component
lives *inside* a quantum until much later in this sequence. You are
labeling parts, not claiming independent deploy.

**Fitness function** — Bounds on component size, checked in CI: e.g.
lines or type-count per component folder between a floor and a
ceiling; or "no component may be a single class, and none may exceed
N types without an ADR." A weaker but useful check: every production
package must be registered in a component map file; unknown packages
fail the build (stops `misc/` from growing).

**Failure if skipped** — You "split" at class grain and get a mesh of
tiny deployables, or you split at layer grain and get
`InvoiceController` talking across the network to `InvoiceService`.
The later patterns will operate on ghosts. Production: a team
extracts `PriceCalculator` as a service because it was a class they
could find; every checkout now pays a hop, and tax still lives in
three other classes they did not identify.

## Gather common domain components

**Problem** — The same business noun is implemented in five places
(`Customer`, `Address`, `Money`, `Site`) *or* dumped into a
`common/` / `shared/` / `util/` junk drawer. You cannot see what is
*truly* shared domain versus what was convenient to import.

**What you do** — **Gather** the duplicated or scattered *domain*
bits into explicit common-domain components — still inside the
monolith. The point is not DRY as a virtue. The point is to make
sharing *visible* so you can later decide, in
[ch. 8](../8-reuse-patterns/) and [ch. 9](../9-data-ownership/):
duplicate, library, or service. Until it is gathered, every future
service will secretly depend on a helper nobody admitted was
architecture.

Rules of thumb while gathering:

- Technical utilities (string pad, clock) are not domain components.
  Keep them out of the domain pile or you will recreate the boulder
  from [ch. 4](../4-decomposition/).
- A noun used by two workflows (customer, site) is a candidate.
  Gathering it is not the same as *extracting* it as a service —
  that is often the zone-of-pain move. You are putting it on the
  map.
- Stop adding random methods to the gathered component. Gathering
  is a conservation move, not an invitation to grow `SharedKernel`.

**Fitness function** — Duplicate-type detection (same record shape
under three names) or a rule: domain types may live only under
`components/<name>` or `components/common-domain/<name>`. Ban new
types in `util/` that are not purely technical (lint on package +
name heuristics, or a denylist of words like `Customer` in `util`).
ArchUnit-style: `util` must not depend on domain packages, and
domain packages must not reach into each other *through* `util`.

**Failure if skipped** — The "common" jar becomes the real system.
Every hoped-for service compiles against it; you have one quantum
with extra logos. Or the opposite: you never gather, so the same
`Status` enum forks six times and no workflow can parse another
workflow's events. Production: identity "helpers" copied into eight
services, then a GDPR deletion that updated only three.

## Flatten components

**Problem** — Nested packages hide the graph. `company.app.billing.internal.impl.legacy.v2`
looks hierarchical. The compiler sees a clique. Component-based work
on a tree you made up is fiction: you will think invoicing depends
on "billing" when it actually imports a class six levels down that
imports dispatch.

**What you do** — **Flatten** so that components sit at one
architectural level. Nesting for file navigation is fine;
*component-of-component* as an architecture story is not. After
flattening, the boxes you named in pattern 1 are the boxes you will
draw arrows between in pattern 4.

```
  before (lie):                    after (visible):
  billing                          billing
    +-- invoices                     invoicing
    +-- tax                          tax
         +-- rules                   dispatch   <-- now an arrow
              +-- dispatch-hook        you can forbid
```

You will find fake parents: an `ops` folder that contained both
dispatch and invoicing because one team owned "operations." Flatten
until the name matches a change-reason, not an org-chart fossil.

**Fitness function** — Maximum package depth *for component roots*
(e.g. component roots live at `components/<name>/...` and
`<name>` is not allowed to contain another registered component).
Or: a generated component graph has no containment edges, only
dependency edges; a test fails if a new nested module is marked as
a component.

**Failure if skipped** — Dependency maps look clean at the parent
level while the nested code reintroduces cycles. You "create a
domain service" that still *contains* the other domain. Production:
`modules/field/` ships as one JAR because nobody flattened
ticketing vs nonticketing; the service boundary is a zip file.

## Determine component dependencies

**Problem** — You still do not know who depends on whom at the grain
you just flattened. Cycles, hidden static coupling, and
"just one import" leaks are how chapter 4's Ca/Ce become surprises
in week eight of an extraction.

**What you do** — Map **component dependencies** explicitly: compile
time, and the cheap static stand-ins for data (which component
touches which tables or types). Mark:

- Direction (who owns the arrow).
- Cycles (break or accept with an ADR; accepting a cycle *across* a
  hoped-for service cut is how you fail).
- Afferent vs efferent at this grain, so leaves and boulders are
  obvious.

This is chapter 4's metrics, now applied to the *named* components
rather than to whatever the IDE called a module.

```
  [intake] --> [assignment] --> [dispatch]
       \                           ^
        \---- [notifications] ----/
                   |
                   v
              [invoicing]     cycle? if invoicing --> assignment, stop
```

Break cycles *before* grouping domains. Typical moves: invert a
dependency onto a contract (both depend on `TicketClosed`, neither
on each other's internals); replace a call with an event you do not
need answered; duplicate a tiny read model rather than importing a
write model. Do not "break" a cycle by adding a third component that
both import — that is how `common` becomes the boulder.

**Fitness function** — ArchUnit / import-linter / dependency-cruiser:
allowed edges live in a file; any other component-to-component
import fails CI. Cycle detection is a red build, not a warning on a
dashboard. Optional: a check that Ca of `common-domain` does not
increase without an ADR.

**Failure if skipped** — You extract along a cycle and buy a
distributed deadlock: neither side can deploy, each waiting on the
other's DTO. Production: "billing" and "tickets" services that must
release in one CR because each still imports the other's package,
now over the network as a required protobuf field.

## Create component domains

**Problem** — Forty honest components are not forty services, and they
are not yet a decomposition target. Without clustering, someone will
pick components at random ("notifications is small, extract that")
and ignore that notifications is an *efferent spider* sitting on
every workflow.

**What you do** — Group components into **component domains**:
clusters that share a change-reason and talk more *inside* the
cluster than outside. This is adjacent to bounded-context thinking
but stays operational: you are clustering for extraction, not
redrawing the enterprise ontology.

In the book's ticketing/field-ops case as a label:

- A **ticketing** domain might gather intake, assignment, dispatch.
- A **nonticketing / commercial** domain might gather invoicing, tax,
  parts.
- **Notifications** might sit with the domain that *owns the moment*
  (ticket closed vs invoice issued) rather than as a global dump.

A component that every domain wants (identity, customer) is not "a
domain called Shared." It is an unresolved ownership problem. Leave
it marked. [Chapter 8](../8-reuse-patterns/) and
[ch. 9](../9-data-ownership/) exist because this step will surface
orphans. Do not "solve" them by extracting Shared as a service in
this chapter.

**Fitness function** — Coupling *between* domains vs *inside*: e.g.
a ratio of cross-domain imports to in-domain imports must stay
below a threshold, or simply: no new edge between domain A and
domain B except through an allowlist (published types/events). A
build-time check that a component's package prefix matches its
declared domain.

**Failure if skipped** — "Services" that are grab-bags (whatever two
engineers were working on that month). You did not decompose; you
shuffled. Production: a `platform-service` that contains scheduling,
PDF, and feature flags because those components were leftover after
the "real" domains were named.

## Create domain services

**Problem** — Domains still live in one deployable. If a
[ch. 3](../3-modularity/) driver truly needs independent deploy,
test, scale, or failure isolation *as another quantum*, you still
have only packages.

**What you do** — Promote a **component domain** into a **domain
service**: its own deliverable (process, pipeline, artifact). Rules
that keep this from becoming a costume:

- No compile-time dependency on another domain's internals. Only
  documented dynamic coupling ([ch. 2](../2-coupling/)).
- You can ship it on a Tuesday without shipping the origin *as a
  requirement*. If you cannot, it is still one quantum — maybe a
  useful module, not a service.
- Data may still be shared *for a while*. If so, the ADR must say
  so, and you may not claim the availability driver is fully bought.
  [Chapter 6](../6-operational-data/) is the next move, not an
  implied bonus of this pattern.

Start with a leaf domain (chapter 4: high I, low Ca), not with the
boulder. Ticketing vs invoicing: extract the one whose graph is a
leaf *and* whose driver is loud. If both are loud and both sit on
`work_orders`, you do not have a domain service yet; you have a
data problem.

**Fitness function** — The new artifact's build forbids the old
monolith as a library. Contract tests at the published edge. A
release check: the service's pipeline can go to production while
the origin's is red (at least in a non-prod environment, as a drill).
Optional: runtime isolation probe — origin down, leaf domain still
serves its read path, if that was the availability claim.

**Failure if skipped** — You stop at domains-in-packages and tell
the business you "decomposed." Deployability and failure isolation
did not move. Or you skip *to* this pattern: a new repo copied from
trunk with all domains still inside — a fork without the earlier
visibility. Production: "invoice-service" that cannot boot without
the monolith module on the classpath, and a pager that still storms
when reporting OOMs the origin.

## Short summary of the sequence

The six patterns are a **single method**. Skipping a step does not
save time; it relocates the unknown.

1. **Identify and size** — get a grain you could extract.
2. **Gather common domain** — put shared nouns on the map so they
   stop hiding in `util`.
3. **Flatten** — make the graph two-dimensional.
4. **Determine dependencies** — arrows, cycles, Ca/Ce; break what
   the future cut cannot survive.
5. **Create component domains** — cluster by change-reason, not by
   leftover parts.
6. **Create domain services** — only if a driver needs a new
   quantum, and only for a leaf you can actually ship.

You can stop after 4 or 5 and still have a more honest modular
monolith. That is a valid outcome of this chapter. The failure is
not "we didn't get to Kubernetes." The failure is claiming a domain
service while step 4 still shows a cycle through a shared kernel.

Tactical forking from [ch. 4](../4-decomposition/) does not replace
this sequence; it postpones it. After a fork, run the same six
moves *inside* the copy so you are not staffing two mudpiles. The
fitness functions above are how [ch. 1](../1-no-best-practices/)
shows up as CI, not as a poster.

What this chapter does *not* do: pick service granularity (that is
[ch. 7](../7-service-granularity/)), split operational data (ch. 6),
or choose library vs sidecar for the leftover common domain
(ch. 8). If those questions showed up while you gathered and
clustered, write them down. Do not solve them by making `common` a
service.

## Check yourself

1. Why is a class the wrong grain for a component, and a technical
   layer the other wrong grain? Give a production extract that
   failed because of grain, and name the failure mode.
2. You skip "gather common domain" and extract invoicing. Where does
   `Customer` live the next morning, and what GDPR-shaped incident
   does that setup cause?
3. Flattening: describe a nested package in a repo you know that
   *looks* like hierarchy and is actually a cycle. What fitness
   function would have caught a new nested "component"?
4. Draw (in words) three component arrows from a system you have
   shipped. Where is the cycle you would have to break before a
   domain service is legal?
5. Why is "extract Shared / Common as the first domain service" the
   zone-of-pain move from chapter 4 wearing chapter 5 clothing?
6. Component domain vs domain service: you stop after clustering.
   Which modularity drivers can improve, and which cannot, without
   step 6?
7. Pick a leaf you would promote first in the book's
   ticketing/field-ops case (label only). What Ca/Ce evidence made
   it a leaf, and what table would still make it one quantum?
8. Write one automated fitness function for *two* different
   patterns in this sequence. What skip-failure does each prevent
   on the next "tiny helper" PR?
9. A team forks (ch. 4) and never runs this sequence on the copy.
   Six months later, what does on-call look like? Be specific about
   duplication and deploys.
10. Someone wants to jump to domain services because "the drivers
    in chapter 3 are all red." Which single pattern in this chapter
    would you force them to finish first, and why is that the
    cheapest honest delay?

Continue to [Pulling apart operational data](../6-operational-data/).

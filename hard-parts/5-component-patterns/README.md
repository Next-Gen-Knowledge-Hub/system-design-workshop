# Component-based decomposition patterns

Software Architecture: The Hard Parts, Ford, Richards, Sadalage, Dehghani, O'Reilly 2021, Chapter 5.

## What the sequence is

Component-based decomposition is one pipeline. The moves run in order: identify and size components, gather common domain components, flatten components, determine component dependencies, create component domains, create domain services. Shuffling them, or skipping to a deployable, extracts the wrong grain, hides coupling inside nested packages, and names a random cluster of classes a domain service.

The moves are code moves. They make a seam you can put in an ADR and a fitness function. An **ADR** is a short dated record of context, one decision, and consequences. A **fitness function** is an automated check that a claimed property still holds. The pipeline does not split the database.

A **component** is a clump of code that changes for one reason and that you might one day deploy apart. A **quantum** is the smallest set of code and data that can be deployed independently and still hangs together. A component lives inside a quantum until a later move in this same sequence promotes a whole domain. Naming components does not claim independent deploy.

**Static coupling** is visible without running the system: imports, shared types, shared tables, a library both sides compile against. **Dynamic coupling** is a runtime link you can document: a call or a message, who waits, who owns the payload. **Afferent coupling (Ca)** counts who depends on a component. **Efferent coupling (Ce)** counts who it depends on. A **boulder** is concrete and widely depended on (high Ca, low abstractness): the zone of pain. A **leaf** has high instability and low Ca. Instability is `I = Ce / (Ca + Ce)`.

Jumping to a domain service because Kubernetes is ready produces a distributed monolith with extra YAML. Stopping after the dependency map can still be a win: maintainability (a person can change one reason without touching the other) and testability (one part can be proven without the rest running) inside one quantum, without a second database.

The examples use a field-ops product. Ticketing is intake, assignment, and dispatch. Nonticketing work is invoicing, tax, and parts. Both touch customers, sites, and `work_orders`.

## Identify and size components

The unit of thought is wrong when classes are the architecture, or "the application" is, or the folders are layers (`api/`, `domain/`, `infra/`) that cut across every change-reason. Nobody can point at what would be extracted.

Name components at the grain a domain expert would point to: intake, assignment, dispatch, invoicing, inventory, notifications. `InvoiceController` and `helpers` are the wrong names. Size the clump so that a single class or a single function does not become a nano-service that still chatters to everyone, and so that half the monolith is not one component that cannot move and cannot be tested as a piece.

A practical component is a package, module, or project whose name is a noun-verb the business uses, holding a handful of related types, with a test target that runs without booting the world. If the drawing has more boxes than about twenty, the list is files, not components.

A CI bound can check size: lines or type-count per component folder between a floor and a ceiling, or a rule that no component is a single class and none exceeds N types without an ADR. A weaker check still helps: every production package is registered in a component map, and an unknown package fails the build. That stops `misc/` from growing.

On this product, assignment is a component. `PriceCalculator` is a class. Extracting the class because it was easy to find puts a network hop on every price, while tax still lives in three other classes that were never named. The later moves then operate on ghosts. The same miss at layer grain puts `InvoiceController` on the network talking to `InvoiceService`.

## Gather common domain components

The same business noun is implemented in five places (`Customer`, `Address`, `Money`, `Site`), or it is dumped into `common/`, `shared/`, or `util/`. You cannot see what is truly shared domain and what was convenient to import.

Gather the duplicated or scattered domain bits into explicit common-domain components, still inside the monolith. The point is to make sharing visible. Visible sharing can later be resolved as a duplicate, a library, or a service. Until it is gathered, every future service secretly depends on a helper nobody admitted was architecture. DRY, as a virtue by itself, is not the point.

Technical utilities (string padding, a clock) are not domain components. Leave them out of the domain pile. Putting them in recreates the boulder: a concrete clump everyone depends on. A noun used by two workflows, such as customer or site, is a candidate to gather. Gathering it puts it on the map. Extracting it as a service is the zone-of-pain move. Stop adding random methods to the gathered component. Gathering conserves what is already shared. It is not a license to grow `SharedKernel`.

Fitness checks: detect the same record shape under three names, or allow domain types only under `components/<name>` or `components/common-domain/<name>`. Ban new types in `util/` that are not purely technical — a lint on package and name, or a denylist that includes words like `Customer` inside `util`. ArchUnit-style rules: `util` does not depend on domain packages, and domain packages do not reach each other through `util`.

Skipped, the common jar becomes the real system. Every hoped-for service compiles against it: one quantum, with extra names on the boxes. Or the noun is never gathered, the same `Status` enum forks six times, and no workflow can parse another workflow's events. Identity helpers copied into eight services, then a GDPR deletion that updates three of them, is that incident.

## Flatten components

Nested packages hide the graph. `company.app.billing.internal.impl.legacy.v2` looks like a hierarchy. The compiler sees a clique. Work done on a tree you invented is fiction: invoicing appears to depend on "billing," and it actually imports a class six levels down that imports dispatch.

Flatten so components sit at one architectural level. Nesting for file navigation can stay. Component-of-component, told as the architecture, cannot. After this move, the boxes from the first pattern are the boxes the dependency map will use.

```
  before (the names lie)           after (the arrows are real)
  billing                          billing
    invoices                         invoicing
    tax                              tax
      rules                          dispatch
        dispatch-hook                  an edge you can forbid
```

Fake parents show up here. An `ops` folder held both dispatch and invoicing because one team owned "operations." Flatten until the name matches a change-reason. An org-chart fossil is not a component.

A fitness function caps depth for component roots: roots live at `components/<name>/...`, and `<name>` may not contain another registered component. Or the generated component graph has no containment edges, only dependency edges, and a test fails when a new nested module is marked as a component.

Skipped, dependency maps look clean at the parent while nested code puts the cycles back. A domain service still *contains* the other domain. `modules/field/` ships as one JAR because ticketing and nonticketing were never flattened. The service boundary is a zip file.

## Determine component dependencies

After the flatten, who depends on whom is still unknown. Cycles, hidden static coupling, and a single import become surprises in week eight of an extraction, which is when Ca and Ce were supposed to be obvious.

Map component dependencies explicitly. Include compile time, and the cheap static stand-ins for data: which component touches which tables or types. Mark direction (who owns the arrow), cycles, and afferent versus efferent at this grain, so leaves and boulders are visible. The counts apply to the named components, not to whatever the IDE called a module.

```
  [intake] --> [assignment] --> [dispatch]
       \                           ^
        \---- [notifications] ----/
                   |
                   v
              [invoicing]
```

If invoicing also points at assignment, stop. That cycle is not a legal service cut.

Break cycles before grouping domains. Invert a dependency onto a contract: both sides depend on `TicketClosed`, and neither depends on the other's internals. Replace a call with an event you do not need answered. Duplicate a tiny read model rather than importing a write model. Adding a third component that both sides import does not break the cycle. That third component is how `common` becomes the boulder.

A cycle across a hoped-for service cut can be broken, or it can be accepted in an ADR. Accepting it, and extracting anyway, is how the cut fails.

Fitness checks: ArchUnit, import-linter, or dependency-cruiser, with allowed edges in a file. Any other component-to-component import fails CI. Cycle detection is a red build, not a dashboard warning. Optionally, Ca of `common-domain` may not increase without an ADR.

Skipped, extraction follows the cycle and buys a distributed deadlock: neither side can deploy, each waiting on the other's DTO. Billing and tickets must release in one change request because each still imports the other's package, now as a required protobuf field on the wire.

## Create component domains

Forty honest components are not forty services, and they are not yet a decomposition target. Without a cluster, someone extracts notifications because it is small, and misses that notifications is an efferent spider sitting on every workflow.

Group components into **component domains**: clusters that share a change-reason and talk more inside the cluster than outside. The clustering is for extraction. It stays operational. Redrawing an enterprise ontology is a different task. Bounded-context language is adjacent; the test here is the import cluster, not the ontology.

On this product:

- A ticketing domain can gather intake, assignment, and dispatch.
- A nonticketing, or commercial, domain can gather invoicing, tax, and parts.
- Notifications sit with the domain that owns the moment — ticket closed, or invoice issued — rather than in a global dump.

A component every domain wants, such as identity or customer, is an unresolved ownership problem. It is not "a domain called Shared." Leave it marked. This step surfaces those orphans. Extracting Shared as a service here is the boulder moved onto the network.

A fitness function compares coupling between domains with coupling inside them: the ratio of cross-domain imports to in-domain imports stays under a threshold, or no new edge between domain A and domain B exists except through an allowlist of published types and events. A build check can require a component's package prefix to match its declared domain.

Skipped, the "services" are grab-bags of whatever two engineers were working on that month. The parts were shuffled. A `platform-service` that contains scheduling, PDF generation, and feature flags is the leftover pile after the "real" domains were named.

## Create domain services

Domains can still live in one deployable. A domain service exists when a driver truly needs another quantum: independent deploy, an isolated test, separate scale, or failure isolation. The drivers are the pains of change, test, release cadence, hardware, and blast radius. Packages alone do not buy those.

Promote a component domain into a **domain service**: its own deliverable, with its own process, pipeline, and artifact. Three rules keep the promotion from being a costume.

- No compile-time dependency on another domain's internals. The only coupling left is documented dynamic coupling.
- It can ship on a Tuesday without shipping the origin as a requirement. If it cannot, it is still one quantum: a useful module, not a service.
- Data may stay shared for a while. The ADR says so. The availability driver is not fully bought while that sharing remains. Splitting the operational data is a separate move. It is not included in this pattern.

Start with a leaf domain: high I, low Ca. Extract the field-ops side whose graph is a leaf and whose driver is loud. If ticketing and invoicing are both loud, and both sit on `work_orders`, there is no domain service yet. There is a data problem.

Fitness checks: the new artifact's build forbids the old monolith as a library. Contract tests cover the published edge. A release drill shows the service pipeline reaching production while the origin pipeline is red, at least in a non-production environment. If the claim was availability, a runtime probe shows the origin down and the leaf domain still serving its read path.

Stopping at domains-in-packages and telling the business the system was decomposed leaves deployability and failure isolation where they were. Skipping *to* this pattern — a new repository copied from trunk with every domain still inside — is a fork without the earlier visibility. An `invoice-service` that cannot boot without the monolith module on the classpath, and a pager that still storms when reporting exhausts memory in the origin, is that outcome.

## After a fork, and what this sequence leaves open

Tactical forking copies a slice and deletes toward something runnable. It postpones this pipeline. It does not replace it. After the fork, run the same six moves inside the copy. Otherwise two mudpiles are on the pager: duplicated bugs, and deploys that still have to line up because nobody flattened or broke the cycles in the copy. The fitness functions above are how the ADR shows up in CI.

You can stop after the dependency map, or after component domains, and keep an honest modular monolith. That is a valid end. The failure is a claim of a domain service while the dependency map still shows a cycle through a shared kernel.

Three decisions stay outside the pipeline. How small a service should be. How operational tables are split. Whether leftover common domain becomes a library or a sidecar. When gathering and clustering surface those questions, write them down. Turning `common` into a service does not answer them.

## What goes wrong

- **Wrong grain.** Class-sized deployables chatter. Layer-sized cuts put a controller on the network in front of its own service. `PriceCalculator` as a service adds a hop, and the unnamed tax classes stay behind.
- **Shared nouns left scattered or dumped in `util`.** One common jar is the real system, or one `Status` enum exists six times and events do not parse. A deletion rule updates three of eight copied identity helpers.
- **Nesting left in place.** Parent-level maps look acyclic. `modules/field/` ships ticketing and nonticketing as one JAR.
- **Cycles carried into the extract.** Neither side deploys. A required protobuf field forces one change request for billing and tickets. A new `common` component, added so both sides can import it, becomes the boulder.
- **Domains that are leftovers.** Notifications extracted because they are small, while they call every workflow. A platform service holds scheduling, PDFs, and feature flags.
- **A service announced too early.** The artifact cannot boot without the monolith on the classpath. Reporting still takes the origin down with it. Or the teams stop at packages and claim that deployability and failure isolation moved.
- **A fork that never enters the pipeline.** Six months on, two copies diverge, fixes land once, and the release train has a second copy of itself.

## Recap

Size components a domain expert would name, between a single class and half the system. Gather shared domain nouns into an explicit common-domain place, and keep technical utilities out of it. Flatten so each component is a root, not a child of another component. Record arrows, Ca, and Ce, and break cycles before any grouping that is meant to deploy apart. Cluster by change-reason into component domains, and leave identity and customer marked until someone owns them. Promote a leaf domain to its own artifact only when a driver needs another quantum, the build no longer links the monolith, and any shared tables are written down as still shared.

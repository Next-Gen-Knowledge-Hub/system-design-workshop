# Architectural decomposition

Software Architecture: The Hard Parts, Ford, Richards, Sadalage, Dehghani, O'Reilly 2021, Chapter 4.

## What decomposition decides

A **driver** is a characteristic that is already bad because too much sits in one place: the system is hard to change, hard to test, hard to ship, hard to scale, or one failure takes down an unrelated workflow. Drivers say why a seam would help. Decomposition asks whether the code will allow a cut, and which extraction style to use.

You cannot extract what you cannot see. **Afferent coupling**, **efferent coupling**, and distance from the **main sequence** tell you whether a seam is real. Then you choose **component-based decomposition** or **tactical forking** on purpose.

A few terms the measures use:

- A **component** is a clump at a grain you might extract. A class is almost never that grain. The whole application is too coarse.
- A **quantum** is the smallest set of code and data you can deploy independently and still have it hang together. High **static coupling** inside it is expected: imports, tables, and internal calls that ship together. Static coupling across a hoped-for cut means the cut is not real yet.
- **Cohesion** means the code and data in that quantum change for one business reason. Driving every internal dependency to zero destroys that.
- An **ADR** is a short dated record: context, one decision, consequences. A **fitness function** is an automated check that a claimed property still holds.

A component that is both concrete and stable — many incoming dependencies, no abstraction — is a trap. Everyone needs it, nobody can change it, and extraction drags the rest of the graph along. If those arrows cannot be measured, the choice of method is a guess.

## When the code is decomposable

A codebase is **decomposable** when you can name components, see how they depend on each other, and find a cut that does not rewrite half the graph in one move. Packages by themselves are not that. Layer packages (`controllers/`, `services/`, `repos/`) often hide the domain seams the drivers pointed at.

Work in this order:

1. Point at components at a useful grain. If the only names are classes or "the app," identification comes before any extraction.
2. List dependencies between those components, including cycles, and including data: tables and shared types.
3. Find **sinks of pain**: concrete clumps that many others depend on, which will poison every cut.
4. Compare the cost of analysis with the cost of copying the mudpile and deleting. When analysis is slower, that comparison chooses forking. It is not a verdict on the team.

If the first or second check fails, the system is not decomposable yet. The work is to make it analyzable. A new repository that still imports the old one is the result of skipping that.

Leadership funds "extract billing" because a driver is real. Engineering starts at the HTTP boundary and finds that every billing call touches `WorkOrder`, `UserContext`, and a static `AppState`. Spend the week on the graph. Language analyzers, ArchUnit, `jdeps`, import graphs, or a spreadsheet of package edges beat intuition. The field-ops target is a component boundary around invoicing. An invoice table existing is a different fact.

"It's pretty clean," with no graph, is a vibe. Six weeks later the new service still compiles against the monolith as a library. That is a distributed build.

## Afferent and efferent coupling

Count at component grain, the grain you might extract. Class grain is the wrong count unless one class actually is the component, which it almost never is.

- **Afferent coupling (Ca)** is how many other components depend on this one. Incoming arrows. High Ca means stable in practice: changing it breaks a crowd. Stable here means hard to change. It does not mean the component runs well.
- **Efferent coupling (Ce)** is how many components this one depends on. Outgoing arrows. High Ce means unstable in practice: it changes when anything it depends on changes.

```
  [intake] --Ce--> [assignment] <--Ca-- [dispatch]
                      ^
                      | Ca
                   [notifications]
```

A dispatch component with high Ce, talking to everyone, is a spider. An identity or `common-domain` component with high Ca is a boulder. Spiders and boulders both resist a clean extraction, and they fail in different ways.

Extracting a boulder — `CustomerHelper`, `BaseEntity`, `SharedKernel` — as a service, because everyone uses it and it "must be a platform," turns the most statically coupled clump into a network dependency for every quantum.

Use Ca and Ce as warnings. High Ca: stabilize it from above with an abstraction, or split the boulder by change-reason before anyone extracts it. High Ce: the component does too much, or it sits at the wrong layer of the graph. Flatten it and cut again.

Optimizing the numbers alone is a different failure. Driving Ce to zero inside a quantum removes the cohesion that quantum needed. Driving Ca to zero on a real domain concept such as `Customer` usually means the concept was copied everywhere. The metric is a lens for cuts. It is not a KPI for a dashboard that HR will gamify.

Cycles are the special case: Ca and Ce become mutual. A cycle between hoped-for billing and hoped-for dispatch means the cut is not in the code. Break it before extraction: a callback, an event, a duplicated read model, or an inverted dependency. Extracting the cycle produces a distributed deadlock of deploys. Neither side can ship, because each is waiting on the other.

## Abstractness, instability, and the main sequence

Two ratios sit on those counts. They are old object-oriented measures. They still work at component grain if the formula is not treated as the goal.

**Instability** is `I = Ce / (Ca + Ce)`.

- I near 0: little outgoing, lots of incoming. Stable. It should be hard to change, so it needs to be a stable abstraction — a contract, an interface, a small published type — rather than a 4,000-line concrete `Utils`.
- I near 1: lots of outgoing, little incoming. Unstable. That fits a concrete workflow at the edge, such as a dispatch worker that calls many things and that almost nobody calls.

**Abstractness (A)** is the fraction of the component that is abstract (interfaces, abstract types, stable contracts) versus concrete implementation. A near 1 is mostly contracts. A near 0 is mostly code that does the work. A practical stand-in, when a perfect analyzer of "abstract" is unavailable, is published APIs and types versus everything else. The relationship between A and I matters. Three decimal places do not.

A concrete `Common` package with I near 0 — everyone depends on it, it depends on almost nothing — forces every extraction to take `Common` along, call it over the network, or rewrite it. That corner is the **zone of pain**. Pain is concrete *and* widely depended on. Concrete code by itself is ordinary.

The other ditch is the **zone of uselessness**: highly abstract and highly unstable (I near 1 and A near 1). A forest of interfaces that depend on everything, and that nobody treats as a stable contract. There is nothing to extract. There is ceremony.

```
  abstract
     ^   zone of uselessness
     |   (abstract and unstable)
     |
     |        *  main sequence
     |        *  A + I ≈ 1
     |
     |   zone of pain
     |   (concrete and stable)
     v
  concrete              stable (high Ca) -->
```

The **main sequence** is the diagonal where `A + I ≈ 1`.

- Abstract and stable: contracts, published events, a tiny identity protocol.
- Concrete and unstable: edge workflows, jobs, UI adapters.

**Distance from the main sequence (D)** is how far a component sits from that diagonal, commonly `|A + I − 1|`. D near 0 means this is a kind of component that can be moved. D near 1 means it is painful or useless.

```
  on the diagonal:
    [TicketClosed contract]    A high, I low    stable abstraction
    [dispatch worker]          A low,  I high   concrete edge

  off it:
    [SharedKernel concrete]    A low,  I low    pain
    [AbstractServiceBase*]     A high, I high   useless
```

Use D as a triage list. Sort components by D, then by Ca. The top of that list is where a decomposition lies: the extract is named "billing," and billing *is* the kernel, so nothing separable moved.

The only candidate for a first service is also the component with the worst D and the highest Ca. Do not start there. Repair it until D drops — split the kernel, introduce a contract, break a cycle — or choose tactical forking so the work is not blocked on a year of purification. Component-based decomposition stays honest when it starts at a **leaf**: high I, low Ca.

If you cannot name the component, the incoming arrow, and the production failure you are afraid of, the plot is not helping. Draw the import graph.

On the field-ops model, `WorkOrder` as a concrete god table or god type, with afferent coupling from intake, assignment, invoicing, and parts, sits in the zone of pain. Extracting `WorkOrder` as a service drags every workflow. Decide which workflow owns which slice, or fork and delete until ownership is forced.

Rewriting the system to improve A and I, with no cut in mind, chases a score. Class-level Chidamber and Kemerer numbers on a polyglot monolith can declare the Java side indecomposable while the real boulder is a shared Postgres enum.

## Component-based decomposition

Identify components, clean their dependencies, group them into domains, and then extract a domain into a new quantum.

You gain three things. The end state matches the domain you meant to isolate. You see the cycles before they become HTTP cycles. Fitness functions can ride along: no new cross-domain imports, and D staying in range for the new boundary.

You pay in time. A mudpile will not become analyzable structure inside one sprint. You also pay in the temptation to polish forever — one more cycle — while the driver keeps charging.

Folder names can look like components when they are layers, and the real graph is a clique. Believe the arrows. If components cannot even be identified, there is no component-based path yet.

The failure is analysis that never cuts, or a "component extraction" that still shares the kernel as a library, so the static coupling never moved.

## Tactical forking

**Tactical forking** copies the codebase, or a large slice of it, into a second deployable, then deletes what that second deployable should not own, until something runnable remains. The first deployable can delete in the other direction later.

You gain speed when a driver is already on fire and a team cannot ship because they are glued to a release train. You gain a path through code that cannot be analyzed cheaply: generated mess, missing tests, a clique of static coupling. You have a second artifact on day one, ugly as it is.

You pay in duplication. Bugs get fixed twice, or they get fixed in one copy only. You pay in drift. The fork is a second mudpile unless someone is paid to delete aggressively and to stop merging "just in case." Shared data is untouched by a fork. Two copies of the code on one schema are still one quantum. A fork with no data plan is a process fork.

Invoicing has to be out before fiscal year-end. The import graph is a hairball. Component-based work would start paying off in Q3, which is too late. Fork, delete until invoicing boots, freeze copy-paste back into the origin, and schedule the deletions as the project. The ADR says the move is tactical, states the expiry, and states how you will know merging from origin can stop.

A temporary fork that becomes a product line — two years, two tax engines, nobody owning the diff — is the debt unpaid. Forking and still compiling against the origin is neither method. Forking is a legitimate move, and it is a debt. Name who pays the ongoing cost of two copies, or do not fork.

## Which method

| | Component-based | Tactical forking |
|---|---|---|
| Needs a readable graph | Yes | No (you will still want one later) |
| Time to a first extra deployable | Slow | Fast |
| End-state cleanliness | Higher if you finish | Only if delete and merge are staffed |
| Risk | Polishing, never cutting | Duplicating, never deleting |
| Data split | Still a separate project | Still a separate project |
| Fits | Visible components, cycles you can break | Mudpile, a deadline, a team to unblock |

Neither method is "how we do microservices." Both can stop at a modular monolith when the drivers in play are maintainability and testability. Another quantum is a further choice.

Use component-based decomposition when:

- You can list components and most arrows without inventing them.
- Cycles are few and breakable.
- The first leaf is obvious: high I, low Ca, a concrete edge on the main sequence.
- The driver can wait through several iterations of cleaning that graph.

Use tactical forking when:

- The graph is a clique, or the boulder is the system.
- A driver is already producing incidents, and no leaf is available to extract.
- A team has to leave the release train now, and deletion is staffed.

A hybrid is common. Fork to get a deployable, then identify and clean components inside the fork so the second mudpile does not freeze. Or go component-based until you hit the kernel, then fork that kernel narrowly, with an expiry in the ADR.

Two engineers can treat the methods as taste and turn the choice into a feud. The schema stays shared either way. Put the choice in an ADR: the graph evidence, the driver, the method, the expiry, and the fitness function. Checks that match a fork: no new references from origin into the fork; the fork's Ca onto `SharedKernel` only decreases; billing tests run on the fork without the monolith classpath.

Choosing a method without choosing a first cut is ceremony. Choosing before you know whether the destination is modules or quanta hides the same gap. If the database stays shared, say so. The approach then covers code deployability only.

On the field-ops code, invoicing as a leaf with a visible package graph can take the component-based path. Invoicing as annotations scattered through `WorkOrderService` starts honestly as fork-and-delete. Either way, `work_orders` as a shared table is still one quantum. The new repository is not the split until the data plan says who owns the rows.

## What goes wrong

Declaring the code decomposable from the folder names, then discovering `WorkOrder`, `UserContext`, and `AppState` on every billing path, produces a service that still compiles as a library against the monolith.

Extracting the boulder (`CustomerHelper`, `BaseEntity`, `SharedKernel`, a concrete `Common`, a god `WorkOrder`) creates a network dependency for every quantum, or it drags the universe into the new deployable. Extracting a cycle creates a deadlock of releases. Extracting the zone of uselessness — abstract bases nobody treats as a contract — extracts ceremony.

Starting at the most important noun, `Customer` or `WorkOrder`, starts at high Ca and bad D. The leaf (high I, low Ca) is the component-based first cut. Repair or fork when the only candidate is the kernel.

Component-based work fails by polishing until the driver has been charging the whole time, or by extracting a component that still links the kernel. Tactical forking fails when the fork is never deleted: two tax engines, bugs fixed twice or once, and a standing merge "just in case." A fork that keeps the origin database has not added a quantum. It can still be worth doing when a team has to leave the train, as long as the data plan and the deletion are named.

A choice made as aesthetics, with no measurement of the graph, pays one of those two bills and leaves the schema shared.

## Recap

Name components and their arrows, including tables and cycles, before announcing an extraction. Treat high Ca plus concrete code as pain, and high A plus high I as useless. Sort by distance from `A + I ≈ 1`, then by Ca, and start at a leaf if you are cleaning toward a cut. Use component-based decomposition when the graph is readable and the driver can wait. Fork, delete, and set an expiry when the graph is a clique and a team is blocked now. Record the method, the first cut, and a check. A shared `work_orders` table keeps the result one quantum until ownership of the rows is decided.

# 4. Architectural decomposition

Companion notes for **Chapter 4** of *Software Architecture: The Hard Parts*
(Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani; O'Reilly,
October 2021).

Chapter 3 said *why* you might split. This chapter asks **whether the
codebase will let you**, and **which extraction style to use**. Skip it
and you will announce a service cut on a ball of mud: cycles you cannot
see, a "utils" component everyone depends on, a fork you never delete
from. The drivers stay real. The seam does not exist yet. Decomposition
is a property of the code plus a method, not a staff meeting.

## The mental model

```
  abstract ^
           |  zone of uselessness
           |   (abstract AND unstable)
           |
  Ce-heavy |            *  main sequence
  (unstable)            *   A + I ~= 1
           |            *
           |            *
           |  zone of pain
           |   (concrete AND stable: everyone
           v    depends on a god class/table)
         concrete              Ca-heavy (stable) -->
```

The one sentence to remember a year from now: **you cannot extract what
you cannot see** — afferent/efferent coupling and distance from the
main sequence tell you if a seam is real, and then you choose
component-based decomposition or tactical forking on purpose.

Two consequences fall straight out of that diagram. First, a component
that is both *concrete* and *stable* (lots of incoming deps, no
abstractions) is a trap: everyone needs it, nobody can change it,
extraction will drag the universe. Second, if you cannot measure those
arrows, you are guessing; guessing is how you pick the wrong
decomposition approach.

## Is the codebase decomposable?

A codebase is **decomposable** when you can name components, see how
they depend on each other, and find a cut that does not require
rewriting half the graph in one move. "We have packages" is not that.
Packages that are layers (`controllers/`, `services/`, `repos/`) often
hide the domain seams chapter 3 wanted.

Ask, in this order:

1. Can we point at **components** at a useful grain (not a class, not
   the whole app)? If not, [ch. 5](../5-component-patterns/) has to
   run *before* any extraction.
2. Can we **see dependencies** between those components (including
   cycles, and including *data*: tables, shared types)?
3. Are there **sinks of pain** — concrete, widely depended-on clumps
   that will poison every cut?
4. Is the graph bad enough that analysis is slower than copying the
   mudpile and deleting? That is the forking question, not a moral
   one.

If (1) or (2) is "no," the system is not decomposable *yet*. The work
is to make it analyzable. Pretending otherwise produces a new repo
that still imports the old one.

**Problem** — Leadership funded "extract billing" because the
[modularity drivers](../3-modularity/) were real. Engineering starts
at the HTTP boundary and discovers every billing call touches
`WorkOrder`, `UserContext`, and a static `AppState`.

**Solution** — Spend a week on the graph, not on Kubernetes. Tools
(language analyzers, ArchUnit, `jdeps`, import graphs, even a
spreadsheet of package edges) beat intuition. The running label is
still the book's ticketing/field-ops case: you are looking for whether
*invoicing* is a component with a boundary, not whether invoices
exist as a table.

**Failure mode** — Decomposability as a vibe ("it's pretty clean").
Six weeks later the new service still compiles against the monolith
as a library. That is not a split. That is a distributed build.

### Afferent and efferent coupling

Two directed counts, at **component** grain (the grain you actually
might extract), not at class grain unless a class *is* the component
— it almost never should be.

- **Afferent coupling (Ca)** — how many *other* components depend on
  this one. Incoming arrows. High Ca means you are *stable in
  practice*: changing you breaks a crowd. "Stable" here means
  *hard to change*, not "runs well."
- **Efferent coupling (Ce)** — how many components *this* one depends
  on. Outgoing arrows. High Ce means you are *unstable in practice*:
  you change when anyone you depend on changes.

```
   [intake] --Ce--> [assignment] <--Ca-- [dispatch]
                       ^
                       | Ca
                    [notifications]
```

A dispatch component with high Ce (talks to everyone) is a spider.
An identity or `common-domain` component with high Ca is a boulder.
Spiders and boulders both resist clean extraction; they fail in
different ways.

**Problem** — You try to extract a boulder (`CustomerHelper`, `BaseEntity`,
`SharedKernel`) as a service because "everyone uses it, it must be a
platform." You have just turned the most statically coupled clump into
a network dependency for every quantum.

**Solution** — Use Ca/Ce as *warnings*, not as a popularity contest.
High Ca: stabilize *from above* with abstractions, or split the boulder
by change-reason before anyone extracts it. High Ce: the component
does too much or lives at the wrong layer of the graph; flatten and
re-cut ([ch. 5](../5-component-patterns/)).

**Failure mode** — Optimizing the numbers in isolation. Driving Ce to
zero inside a quantum destroys the cohesion [ch. 2](../2-coupling/)
wanted. Driving Ca to zero on a real domain concept ("Customer")
usually means you duplicated it everywhere. The metric is a lens for
*cuts*, not a KPI for a dashboard that HR will gamify.

Cycles are a special case: Ca and Ce become mutual. A cycle between
hoped-for billing and dispatch means **the cut you wanted is not in
the code**. Break the cycle (callback, event, duplicate a read model,
invert a dependency) before you extract. Extracting a cycle gives you
a distributed deadlock of deploys.

### Abstractness and instability

Two ratios sit on top of those counts. They are old object-oriented
metrics; they still work if you read them at component grain and do
not worship the formula.

- **Instability (I) = Ce / (Ca + Ce)**
  - **I near 0:** little outgoing, lots of incoming. Stable. Should be
    *hard* to change, so it had better be a stable *abstraction* (a
    contract, an interface, a small published type), not a 4,000-line
    concrete `Utils`.
  - **I near 1:** lots of outgoing, little incoming. Unstable. Fine for
    a concrete workflow at the edge (a dispatch worker that calls
    many things and almost nobody calls it).
- **Abstractness (A)** — fraction of the component that is
  abstract (interfaces, abstract types, stable contracts) versus
  concrete implementation. **A near 1:** mostly contracts. **A near 0:**
  mostly code that does the work.

You do not need a perfect static analyzer of "abstract." A practical
stand-in: published APIs and types vs everything else. The point is
the *relationship* between A and I, not three decimal places.

**Problem** — A concrete `Common` package with I ≈ 0 (everyone depends
on it, it depends on almost nothing). Every extraction has to take
`Common` along, or call it over the network, or rewrite it. That is
the **zone of pain**.

**Solution** — Either *abstract* the boulder (depend on a small
contract; hide the concrete mess behind it and stop adding random
helpers) or *break* it by domain so Ca falls. Pain is not "has
concrete code." Pain is **concrete + widely depended-on**.

The other ditch is the **zone of uselessness**: highly abstract *and*
highly unstable (I ≈ 1, A ≈ 1). A forest of interfaces that depend on
everything and that nobody treats as a stable contract. You cannot
extract it because there is nothing to extract — only ceremony.

**Failure mode** — Rewriting the system to pretty-up A and I without
a cut in mind. Metrics exist to find the *next* seam, not to win a
score. Also: applying class-level Chidamber & Kemerer numbers to a
polyglot monolith and declaring the Java side "indecomposable" while
the real boulder is a shared Postgres enum.

### Distance from the main sequence

The **main sequence** is the diagonal where **A + I ≈ 1**:

- Abstract and stable (contracts, published events, a tiny identity
  protocol).
- Concrete and unstable (edge workflows, jobs, UI adapters).

**Distance from the main sequence (D)** is how far a component sits
from that diagonal, commonly `|A + I − 1|`. D near 0 is "this
component is a kind of thing we know how to move." D near 1 is "this
component is either painful or useless."

```
  want:
    [TicketClosed contract]     A high, I low   (stable abstraction)
    [dispatch worker]           A low,  I high  (concrete edge)

  trouble:
    [SharedKernel concrete]     A low,  I low   (pain)
    [AbstractServiceBase*]      A high, I high  (useless)
```

Use D as a **triage list**, not as a grade. Sort components by D,
then by Ca. The top of that list is where decomposition will lie to
you: you will think you extracted "billing" and you actually extracted
nothing because billing *is* the kernel.

**Problem** — The only candidate for a first service is also the
component with the worst D and the highest Ca.

**Solution** — Do not start there. Either repair it (split the kernel,
introduce a contract, break a cycle) until D drops, or choose
**tactical forking** so you are not blocked on a year of purification.
Starting with a leaf (high I, low Ca) is how component-based
decomposition stays honest.

**Failure mode** — "Main sequence" as architecture astrology. If you
cannot name the component, the incoming arrow, and the production
failure you fear, put the plot away and draw the import graph.

For the book's ticketing/field-ops case: if `WorkOrder` is a concrete
god table/type with Ca from intake, assignment, invoicing, and parts,
you are looking at pain. You do not "extract WorkOrder as a service."
You decide which workflow *owns* which slice, or you fork and delete
until ownership is forced. That decision is this chapter plus
[ch. 9](../9-data-ownership/).

## Component-based decomposition vs tactical forking

Once the graph is visible, you still pick a **method** for getting
code into a new deployable. These two are not ideologies. They are
trade-offs with different failure modes.

### Component-based decomposition

You identify components, clean their dependencies, group them into
domains, and *then* extract a domain into a new quantum. [Chapter
5](../5-component-patterns/) is the pattern sequence. This chapter
only places the bet.

You gain:

- The end state matches the domain you meant to isolate.
- You learn the cycles *before* they become HTTP cycles.
- Fitness functions (no new cross-domain imports, D staying in range
  for the new boundary) can ride along.

You pay:

- Time. Analyzable structure is a prerequisite; a mudpile will not
  yield it in a sprint.
- Temptation to polish forever ("just one more cycle") while the
  driver from [ch. 3](../3-modularity/) keeps charging.

**Problem** — The code already has names that look like components,
but they are layers, and the real graph is a clique.

**Solution** — Believe the arrows, not the folders. If the pattern
sequence in chapter 5 cannot even *identify* components, you do not
have a component-based path yet.

**Failure mode** — Analysis paralysis dressed as architecture. Or a
"component extraction" that still shares the kernel as a library, so
static coupling never moved.

### Tactical forking

You **copy** the codebase (or a large slice) into a second deployable,
then **delete** what that second deployable should not own, until
something runnable remains. The first deployable may delete in the
other direction, later.

You gain:

- Speed when the driver is on fire (a team cannot ship because they
  are glued to a release train).
- A path through code that *cannot* be analyzed cheaply: generated
  mess, missing tests, a clique of static coupling.
- An honest second artifact on day one, even if it is ugly.

You pay:

- Duplication. Bugs get fixed twice or not at all.
- Drift. The fork is a second mudpile unless someone is paid to
  *delete aggressively* and to stop merging "just in case."
- Shared data is *not* solved by a fork. Two copies of the code on
  one schema is still [one quantum](../2-coupling/). Forking without
  a data plan is a process fork, not an architecture split.

**Problem** — You needed invoicing out before fiscal year-end. The
import graph is a hairball. Component-based work would start paying
off in Q3.

**Solution** — Fork, delete until invoicing boots, put a freeze on
copy-paste back into the origin, and schedule the deletions as the
actual project. Record an ADR: this is tactical, this is the expiry
date, this is how we will know we can stop merging from origin.

**Failure mode** — "Temporary" forks that become product lines. Two
years, two tax engines, no owner for the diff. Or forking *and*
continuing to compile against the origin, which is neither method.

Tactical forking is a legitimate architecture move. It is not a
confession. It *is* a debt instrument. If you cannot name who pays
the interest, do not sign.

### Trade-offs, side by side

| | Component-based | Tactical forking |
|---|---|---|
| Needs a readable graph | Yes | No (you will still want one later) |
| Time to first extra deployable | Slow | Fast |
| End-state cleanliness | Higher if you finish | Only if delete/merge is staffed |
| Risk | Polishing, never cutting | Duplicating, never deleting |
| Data split | Still a separate project | Still a separate project |
| Fits | Visible components, cycles you can break | Mudpile, deadline, "get a team unblocked" |

Neither approach is "how we do microservices." Both can stop at a
**modular monolith** if [ch. 3](../3-modularity/) only needed
maintainability and testability. Extra quanta are a later choice.

## Choosing a decomposition approach

Choose with evidence from this chapter, not with team identity
("we are a platform org, we don't fork").

Use **component-based** when:

- You can list components and most arrows without lying.
- Cycles are few and breakable.
- The first extractable leaf is obvious (high I, low Ca, sits on the
  main sequence as a concrete edge).
- The driver can wait for a few iterations of chapter 5.

Use **tactical forking** when:

- The graph is a clique or the boulder *is* the system.
- A driver is already an incident factory and a leaf extraction is
  not available.
- A team must get off the train *now*, and you can staff deletion.

Use a **hybrid** more often than either camp admits: fork to get a
deployable, then run component patterns *inside* the fork so the
second mudpile does not freeze. Or: component-based until you hit the
kernel, then a narrow fork of that kernel with an expiry ADR.

**Problem** — Two senior engineers pick methods as aesthetics.
Component-based vs fork becomes a culture war. The schema stays
shared either way.

**Solution** — Put the choice in an ADR ([ch. 1](../1-no-best-practices/)):
graph evidence, driver, method, expiry, fitness function. Examples of
checks: "no new references from origin into fork," "fork's Ca to
`SharedKernel` only decreases," "billing tests run on the fork without
the monolith classpath."

**Failure mode** — Choosing an approach without choosing a *first
cut*. Method-without-seam is ceremony. Also: choosing before you know
whether you are decomposing toward modules or toward quanta. If you
still share the database, say so; the approach then is about *code*
deployability only.

For the book's ticketing/field-ops case: if invoicing is a leaf with
a visible package graph, component-based is available. If "invoicing"
is annotations scattered through `WorkOrderService`, fork-and-delete
is the honest start. Either way, `work_orders` as a shared table
remains the quantum problem. Do not congratulate the new repo until
[ch. 6](../6-operational-data/) has a sentence.

## Check yourself

1. On a codebase you know, is it decomposable *this quarter*? Which of
   the four questions (grain, arrows, pain sinks, fork vs analyze)
   failed, and what would a premature extract look like in production?
2. Pick one component. Roughly: is Ca or Ce winning? What extraction
   mistake does that number warn you about?
3. Explain the zone of pain without the word "metric": a concrete
   thing everyone imports. Give a real `Utils` / `BaseEntity` /
   `SharedKernel` story and what happened when someone "extracted"
   it.
4. Zone of uselessness: what does an abstract-and-unstable component
   look like in a PR review, and why can you not extract it?
5. Why start with a leaf (high I, low Ca) rather than with the most
   "important" domain noun? What failure mode hits if you start with
   `Customer` or `WorkOrder`?
6. Component-based vs tactical forking: pick a system you have seen
   split (or fail to). Which method did they *actually* use, and which
   cost on the comparison table showed up?
7. A fork still uses the origin's database. How many quanta do you
   have, and which chapter's vocabulary answers that? Why is the fork
   still possibly worth it?
8. What fitness function would you attach to a tactical fork so it
   cannot become a permanent duplicate tax engine?
9. Distance from the main sequence: sort three real packages you know
   into "probably movable," "pain," and "useless." What evidence did
   you use besides a formula?
10. You must choose an approach on Friday for the book's
    ticketing/field-ops case (label only). What *one* measurement
    would you take this week to decide fork vs component-based, and
    what would "we chose on aesthetics" have cost?

Continue to [Component-based decomposition patterns](../5-component-patterns/).

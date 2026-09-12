# 15. Build your own trade-off analysis

Companion notes for **Chapter 15** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

Fourteen chapters were **worked examples** of one skill: a decision
that looks like a tool choice is usually several **entangled
dimensions**, and the job is to name them, name the **coupling
points**, and pick in public. This chapter is the skill without a
new pattern catalog. Skip it and you will own a glossary (quantum,
saga, stamp, mesh) and still argue from slogans in the next design
review.

There is no Hard Parts chapter 16. The sequel is your system.

## See also (do not merge)

**Forks in data systems** (OLTP/OLAP, cloud, single-node vs
distributed) open [DDIA ch. 1](../../ddia/1-architecture-tradeoffs/).
This chapter is **how you analyze any seam** — including ones DDIA
never named. [HP 1](../1-no-best-practices/) told you there are no
best practices and gave you ADRs and fitness functions. This
chapter is the **method** those artifacts are supposed to contain.
Do not paste DDIA's four forks into an ADR and call it a Hard
Parts analysis.

## The mental model

```
  "Should we do X?"
           |
           v
  [ 1 untangle dimensions ]     it was four questions
           |
           v
  [ 2 list coupling points ]    who changes together?
           |
           v
  [ 3 relevant cases ]          our plots, not Netflix's
           |
           v
  [ 4 qualitative / numbers ]   do not fake either
           |
           v
  [ 5 bottom line ]             ADR: pick, costs, follow-ups
           |
           x  slogans, snake oil, 40-cell matrices that
              conclude "it depends" and stop
```

The one sentence to remember: **the method is the deliverable** —
a named choice with named costs beats a museum of options.

Two consequences. First, pattern catalogs (including this book's)
are **inputs** to the method, not outputs. Reciting Fairy Tale vs
Time Travel without a coupling analysis is trivia.
Second, "it depends" is the *start* of the work. The output is
"it depends on *these* dimensions; we picked *this* because *that*
cost is the one we can pay."

## Finding entangled dimensions

**Problem** — A design question arrives as a boolean. Sync or
async? Orchestrate or choreograph? Warehouse or mesh? Postgres or
document store? The boolean is a **bundle**.

**Solution** — Split the bundle before you vote. Example: "Should
this plot be choreographed?" hides at least:

- who owns workflow **state** ([ch. 11](../11-distributed-workflows/)),
- whether half-writes may be **visible** ([ch. 12](../12-transactional-sagas/)),
- how fat the **event** is ([ch. 13](../13-contracts/)),
- whether the steps are one **quantum** or four ([ch. 2](../2-coupling/),
  [ch. 7](../7-service-granularity/)),
- which **team** gets the page.

Those are different decisions. They can land on different sides.
Hybrid ("orchestrate money, choreograph notify") is what
untangling is *for*. If your matrix has one row, you have not
started.

How to find dimensions: walk the customer plot, mark **failure**,
mark **change** (who ships a new step), mark **scale** (which box
gets hot), mark **ownership** (who may write). Each mark that is
not forced by the others is a dimension. If two marks always move
together in *this* domain, they are one dimension here even if the
book listed them apart.

**Failure** — Debating the boolean for an hour. The loudest
dimension wins (usually latency or "decoupling") and the silent
ones (undo story, stamp, on-call) show up in production. Also:
copying this book's dimensions as a checklist without asking
whether *this* decision has them. Mesh vs warehouse does not need
a saga cell. A saga cell does not need a column-store argument.

## Coupling: analyze the points

**Problem** — Trade-off tables that never mention **who must
change together** are product comparisons, not architecture.

**Solution** — For each option, list **coupling points**: seams
where knowledge, deploy, or failure crosses a quantum
([ch. 2](../2-coupling/)).

Ask, per seam:

- What **knowledge** crosses (fields, plot order, identity)?
- What **runtime** crosses (sync call, event, batch port)?
- Who **deploys** when this knowledge changes?
- What **fails** if the other side is down or stale?
- Is this coupling **static** (you cannot even start without them)
  or **dynamic** (you couple when the plot runs)?

Then compare options **on the same points**, not on vibes.
Orchestration vs choreography: the conductor is an honest static
hub; the event graph is a dishonest semantic hub. Stamp vs thin
event: you moved coupling from payload shape to extra reads.
Mesh vs warehouse: you moved coupling from a central model to
federated contracts. There is no "uncoupled." There is **coupled
where we can stand it**.

Fitness functions from [ch. 1](../1-no-best-practices/) belong
here: if "ticket-open stays under 300 ms" or "Billing can deploy
without Ticket" is a real constraint, write a check. Analysis
that cannot become a test or an ADR clause is a conversation you
will re-have.

**Failure** — Counting topics, services, or repositories as a
coupling metric. Counting is easy and mostly wrong. A single
stamped event can couple more teams than five tiny commands.

## Assess trade-offs

**Problem** — After untangling, you have a pile. Piles do not
ship.

**Solution** — For each surviving option, write **what you gain**
and **what you pay** on the dimensions you kept. Drop dimensions
that do not move (if both options are eventually consistent, stop
talking about atomicity). Rank costs by **who pays**: customer,
on-call, the team that cannot ship, the team that cannot understand.

A trade-off is assessed when a reasonable colleague can **disagree
on the pick** and still agree on the costs. If they cannot even
agree what the costs *are*, you are still entangled.

Use the catalogs as **menus**, not as answers: saga cells, reuse
patterns, access patterns, warehouse/lake/mesh. Menus prevent
"we invent a ninth cell by accident." They do not pick the lunch.

**Failure** — A 20×20 matrix with traffic lights that ends in
yellow everywhere. That is fear in spreadsheet form. Also:
assessing only the happy path. Partial failure
([DDIA 9](../../ddia/9-distributed-trouble/) for the physics;
[ch. 12](../12-transactional-sagas/) for the plot) is usually
where options diverge.

## Techniques

### Qualitative versus quantitative

**Problem** — Some costs are milliseconds and dollars. Some are
"this team will not understand the rewind." People either invent
fake numbers for the second kind or refuse to measure the first.

**Solution** — **Quantitative** when a number would change the
pick: p99 of the plot, cost of a stamp at planned fanout, abort
rate, freshness SLO. **Qualitative** when the unit is judgment:
cognitive load, irreversibility, regulatory shame. Do not score
"team sanity" as 3.7. Do write the sentence. Do not leave
"latency" as a feeling if you have traces.

**Failure** — Weighted scoring models that launder a gut pick
through invented weights. If you already know the pick, write the
ADR. If you do not, a fake formula will not discover it.

### MECE lists

**Problem** — Options overlap ("events" vs "async") or have a
hole ("we never listed merge the services").

**Solution** — **MECE**: mutually exclusive, collectively
exhaustive. Options should not be the same idea in two costumes.
The set should include **do nothing**, **merge**, and **not our
problem** (change the product rule) when those are real. Dimensions
should not double-count (do not list "coupling" and "independent
deploy" as if they were independent if in this case they are the
same knob).

MECE is a **hygiene** check, not a proof. If a colleague immediately
names a missing option, you were not exhaustive.

**Failure** — Exhaustive lists of *tools* (Kafka, Rabbit, HTTP,
gRPC) when the exclusive axis was *style*. You will pick a tool
and still not have a design.

### The out-of-context trap

**Problem** — "Netflix does it." "The paper says." "Best practice."
[Chapter 1](../1-no-best-practices/) already killed best practices;
they come back wearing conference badges.

**Solution** — Steal **dimensions and failure modes**, not
conclusions. Ask what quantum, org shape, scale, and compliance
regime the source had. If you do not know, you cannot import the
pick. Their choreography assumed a platform team you do not have.
Their warehouse assumed a finance close you *do* have.

**Failure** — Copying a saga cell from a talk whose plot was
optional notifications, onto a plot that moves money. Copying mesh
from a company with fifty domains into a company with two.

### Model relevant domain cases

**Problem** — Abstract "Service A / Service B" makes every option
look fine. Real plots have irreversible steps, humans in the loop,
and a Support org.

**Solution** — Analyze on **your** cases: ticket-open, refund,
tech-no-show, "customer merged two accounts." Walk happy, timeout,
and compensate. If an option dies on a case you actually run, it
is dead, however pretty it is on a generic slide. The book's
ticketing firm was a method: **keep the plot, change the
architecture under it**, see what breaks.

**Failure** — One toy case. Especially the happy path of checkout.
The interesting cell is the 3 a.m. one.

### Prefer the bottom line over overwhelming evidence

**Problem** — Analysis as a pile of appendices. Stakeholders cannot
find the pick. Authors hide because a clear sentence can be wrong
in public.

**Solution** — Lead with: **the choice, the one or two decisive
costs, what you will watch.** Put the matrix in an appendix. An
ADR ([ch. 1](../1-no-best-practices/)) that cannot be quoted in a
standup is too long. Overwhelming evidence is how snake oil looks
"thorough" and how scared architects look "rigorous."

**Failure** — Thirty pages, no sentence that starts with "We will."
A conclusion that restates the options.

### Avoid snake oil and evangelism

**Problem** — Vendors and movements sell **absence of trade-offs**.
Microservices will decouple you. Events will decouple you. Mesh
will scale you. 2PC will make it ACID again. A new encoding will
make contracts safe.

**Solution** — Every sentence that contains "always" or "never"
about an architecture style is a candidate for the bin. Ask what
the pitch **hides** (usually: a platform team, a visible
inconsistency, a stamp, a central bottleneck moved not removed).
Use this book's catalogs as **named costs**, which is the opposite
of evangelism.

**Failure** — Replacing last year's slogan with this book's
vocabulary without doing the analysis. "We picked Anthology"
without a join rule is still snake oil. You just used a nicer
name.

## Short epilogue: the method is the deliverable

The patterns were never the point. Epic and Fairy Tale, stamp
coupling, data product quanta — those are **examples of naming
costs**. You will meet decisions this book does not have a cute
name for (a new compliance rule, a vendor you cannot leave, a
team that does not exist yet). If you can untangle, list coupling
points, run *your* cases, and write a bottom line, you finished
the book. If you can only recite the cube, you finished the
glossary.

The workshop index is the map back into both tracks when a word
("transaction," "consistency," "schema," "workflow," "OLAP")
shows up again and tries to merge folders. Keep them split. Do
the analysis on the job you actually have.

## Check yourself

1. Take a boolean you argued last month. Split it into at least
   three dimensions. Which one actually drove the pick?
2. What is a coupling point that is not "a network call"? Give
   one from contracts and one from analytical data.
3. Why is "number of topics" a bad coupling metric?
4. Name a cost you should quantify and a cost you should not
   pretend to quantify. What goes wrong if you swap them?
5. Write three options for a plot that must charge then assign,
   and show they are MECE. Include "merge the services."
6. An article says "always choreograph." What context would you
   need before the sentence is even *evaluable*?
7. Why do generic A/B services make every saga cell look
   acceptable? What cases would you substitute?
8. Open an ADR you have seen. Can you quote the bottom line in
   one breath? If not, what was being hidden?
9. Pick a slogan from this book ("Fairy Tale," "data mesh,"
   "strict contracts") and state the cost it hides if you use it
   as a conclusion.
10. After this chapter, what is the artifact you owe a design
    review — a catalog, or a method? What does the artifact
    contain?

Continue with the workshop map: [topic index](../../INDEX.md)
(both tracks, same words, different jobs) and the
[DDIA track](../../ddia/README.md) if you still need how the
*stores* behave. There is no next Hard Parts chapter. The next
decision is yours.

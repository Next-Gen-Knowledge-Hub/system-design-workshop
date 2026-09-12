# 1. What happens when there are no "best practices"?

Companion notes for **Chapter 1** of *Software Architecture: The Hard Parts*
(Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani; O'Reilly,
October 2021).

This chapter is the method for the whole Hard Parts track. Distributed
design is not a catalog of winning recipes. It is a habit of naming the
trade-off, writing down why you picked this side, and attaching a check
that will fail when the pick goes stale. Skip it and the rest of the
book becomes a list of patterns you will apply because a blog said so —
then you will be surprised when the shared database, the sync call chain,
or the "temporary" fork is still there two years later.

**See also (do not merge):** [DDIA ch. 1](../../ddia/1-architecture-tradeoffs/)
is the *forks of a data system* (OLTP vs OLAP, cloud vs self-host,
distributed vs single-node). This folder is ADRs, fitness functions, and
the decision *process* when nobody can hand you a best practice. Same
word "trade-off," different job.

## The mental model

```
  "what should we do?"
           |
           v
     best practice? ---- no. there isn't one that fits.
           |
           v
     name the trade-off (what you gain / what you pay)
           |
      +----+--------------------+
      |                         |
      v                         v
  write an ADR              attach a fitness function
  (context, pick,           (how CI or prod will tell
   consequences)             you the pick still holds)
```

The one sentence to remember a year from now: **architecture is the set
of decisions that are expensive to reverse**, and with no portable
"best," those decisions only stay honest if you record the *why* and
measure the *still true*.

Two consequences fall straight out of that diagram. First, "we always
use Kafka / Kubernetes / one service per table" is not architecture; it
is an unevaluated habit. Second, a decision with no ADR and no fitness
function will be relitigated every quarter, or worse, treated as physics
by people who were not in the room.

## Why "the hard parts"

The easy parts of architecture have recipes: how to terminate TLS, how
to take a database backup, how to put a health check on a process. The
hard parts are the questions where every option is wrong for someone in
the room.

- Do we split this monolith now, or pay the coordination tax another
  year?
- Do two workflows share a table, or do we copy data and accept lag?
- Is a synchronous call simpler than an event, or have we just coupled
  two deploy schedules?

Those are not implementation details. They are structural bets. This
track exists because the bets interact: a granularity choice in
[ch. 7](../7-service-granularity/) is also a data-ownership choice in
[ch. 9](../9-data-ownership/) and a saga choice in
[ch. 12](../12-transactional-sagas/). There is no "correct service cut"
that you can paste from another company.

**Problem** — A design review asks for "the industry best practice."
The room picks a pattern by popularity. Six months later the pattern is
the incident.

**Solution** — Treat every structural pick as a trade-off with a named
winner and a named loser. "We accept slower billing deploys so that
dispatch can ship daily" is a decision. "Microservices" is not.

**Failure mode** — You collect practices instead of consequences. The
architecture diagram looks modern. The coupling is the same as last
year.

### Timeless advice

A few statements survive the framework of the month. They are not
slogans to print on a mug; they are filters for the rest of this book.

- There is no decontextualized "best." A practice is a fit for a
  *situation*: team skill, data gravity, regulatory blast radius, how
  often this bit changes.
- You cannot maximize every characteristic at once. Independent deploy
  fights a single ACID transaction. Elastic scale fights a chatty
  in-process call. Write the conflict down; do not pretend it isn't
  there.
- The irreversible (or merely miserable-to-reverse) decisions deserve
  more ceremony than the reversible ones. Database identity, public
  contracts, and "who owns this row" outlive class names.

Timeless is not the same as vague. "It depends" is incomplete until you
say *on what*. The rest of this chapter is the machinery for that
sentence: ADRs capture the dependence, fitness functions keep it from
rotting, and a shared running case keeps later chapters comparable.

### Importance of data in architecture

Code moves. Data sits. A function can be copied into a new service in
an afternoon. A table that three workflows join on will still be there
after the reorg, the rewrite, and the "temporary" dual-write.

When people say they split a system and then wonder why two "services"
must still release on the same Friday, look at the schema first. Shared
rows, shared sequences, shared "status" enums, and foreign keys that
cross a hoped-for boundary are architecture, not persistence trivia.
This track will spend [ch. 6](../6-operational-data/) and
[ch. 9](../9-data-ownership/) on that gravity. Chapter 1 only needs you
to stop designing as if the class diagram were the system.

Consequences you can use immediately:

- If two teams cannot change a column without a meeting, they do not
  have independent architecture, whatever the repo count.
- Analytical copies, search indexes, and caches are part of the
  architecture even when they are "just derived." They constrain
  deletion, privacy, and what "done" means for a write.
- A data model that was convenient for one deployable becomes a
  distributed transaction the moment you cut the process in half.

DDIA's first chapter is *which kind of data system*. This paragraph is
*data as the thing that makes a service cut real or fake*. Keep them
apart.

## Architectural Decision Records

An **ADR** is a short, dated note that records one architecture-significant
choice: the context you had, what you picked, and what you accepted as
the downside. It is not a design spec, not a wiki novel, and not a
ticket comment that will be deleted in the next tracker migration.

**Problem** — The people who chose the shared database left. The new
team treats the sharing as a law of nature, or they rip it out without
knowing which reports, locks, and night jobs depend on the join.

**Solution** — Write ADRs for the decisions whose reversal would take
weeks or would strand data. A usable shape, kept boring on purpose:

1. **Title** — the decision, in a sentence. "Billing and dispatch stay
   on one Postgres until we have a customer identifier we can replicate."
2. **Status** — proposed, accepted, superseded. ADRs die in public;
   they are not silently edited into a different past.
3. **Context** — forces, constraints, options you considered. Enough
   that a new hire can see *why this was hard*.
4. **Decision** — what you will do. One pick, not a menu.
5. **Consequences** — good and bad. The bad ones are the point. If you
   cannot name a downside, you have not finished the trade-off.

Keep them in the repo, next to the code they constrain, numbered, and
grepable. Review them in the same change that implements the decision.
Supersede them with a new ADR when the context changes; do not rewrite
history.

What does *not* need an ADR: a library version, a class rename, a local
refactor that does not move a boundary. Ceremony that fires on every
pull request trains people to skip the ceremony that matters.

**Failure mode** — ADRs as bureaucracy: templates with twenty empty
headings, or a Confluence graveyard nobody reads. Equally: no ADRs at
all, so every incident starts with archaeology. The failure is not
"wrong template." It is *decisions that cannot be found when the
trade-off turns*.

Production picture: a payments team records "we will dual-write to the
ledger and the card processor, and treat the ledger as source of truth
on mismatch." Eighteen months later a new orchestrator is proposed.
The ADR is the artifact that says whether you are allowed to, and what
fitness check must move with the change.

## Architecture fitness functions

A **fitness function** is an automated (or at least scheduled) check
that an architectural characteristic you care about still holds. Think
of it as a unit test for a *property of the system*, not for a method.

The idea is borrowed from evolutionary architecture and then used as a
working tool: you do not "have modularity" because a slide says so.
You have it if a check fails when someone imports across a forbidden
boundary, or when p99 of assign-technician crosses the line you wrote
in the ADR.

**Problem** — Architecture is enforced by memory and code review. Both
fail on Fridays and after headcount changes. The first violation of
"don't join tickets to invoices in the request path" becomes the
example everyone copies.

**Solution** — For each characteristic you claimed in an ADR, attach
something that can go red:

- **Static / structural:** ArchUnit, import-linter, dependency-cruiser,
  a CI job that fails on a cycle between `billing/` and `dispatch/`.
  Cheap. Catches the "I'll just import that helper" leak.
- **Runtime / operational:** a synthetic probe or SLO burn for "dispatch
  p99 < 200 ms," a chaos experiment that billing can be down without
  killing intake, a contract test against a published schema.
- **Pipeline gates vs continual:** some checks run on every merge
  (cycles, forbidden tables). Some run in production all week (error
  budget, queue lag). Do not pretend a nightly dashboard is a merge
  gate.

"Using them" means they are in the path of delivery. A wiki page of
intended fitness functions is a wish. A red build is a fitness
function.

Start smaller than you think. One cycle check and one "this module
must not touch that table" rule will teach the org more than a
framework with forty empty hooks. Add a check when an ADR claims a
characteristic, not when a conference talk lists categories
(atomic vs holistic, triggered vs continual). The categories are
useful for *coverage* — you notice you have only static checks — not
for filling a bingo card.

**Failure mode** — Fitness theater: flaky checks people mute, metrics
nobody owns, or a gate so strict that teams route around it with a
shared "utils" jar. Also the opposite: no checks, so the architecture
you drew on day one is fan fiction by month six.

Production picture: after an incident where a reporting query locked
`work_orders`, a fitness function bans that join from the OLTP
connection role. The next dashboard author hits a red pipeline instead
of a Saturday outage. That is fitness as an operational control, not as
architecture poetry.

## Architecture vs design

Keep the definitions simple enough to use in a review, and refuse to
spend the meeting on taxonomy.

- **Architecture** — the decisions that are expensive to reverse:
  boundaries, communication style, data ownership, the shape of
  deployables, the characteristics you are optimizing (deployability,
  scale, availability). You notice architecture when changing your mind
  means migrating data, versioning a contract, or coordinating three
  teams.
- **Design** — the decisions inside a boundary that a team can redo
  without asking the rest of the company: class structure, local
  caching, which helper library, how a module names its functions.

The line is fuzzy on purpose. A "small" interface choice becomes
architecture the moment two other quanta depend on it. A "big"
diagram that only one team will ever implement is design with extra
boxes.

**Problem** — Architects design class hierarchies; engineers make
irreversible data cuts in a pull request with no ADR.

**Solution** — Ask: *what happens if we are wrong for a year?* If the
answer is "rename and move on," it is design. If the answer is "we run
two systems of record and a reconciliation job," it is architecture —
write the ADR, attach the fitness function, and put it on this track's
map.

**Failure mode** — Architecture as ivory tower (structure without
delivery) or design as accidental architecture (every merge can add a
cross-service join). Both produce systems that cannot be changed on
purpose.

This split also tells you what belongs in later chapters. Component
patterns ([ch. 5](../5-component-patterns/)) are a *method for finding*
architecture. Whether a function is recursive is design.

## Introducing the running case

The rest of this book needs one company to hang decisions on, otherwise
every chapter invents a new domain and the trade-offs cannot be
compared. The notes here use **the book's ticketing/field-ops case**
as a *label* for that shared example — not as a plot to memorize.

You should steal the *idea* of a shared case for your own notes: pick
one real system you have shipped (or one you are about to split) and
reuse it through [ch. 15](../15-trade-off-analysis/). If you switch
companies every chapter, you will never see how a coupling choice in
[ch. 2](../2-coupling/) becomes a saga in [ch. 12](../12-transactional-sagas/).

### Nonticketing vs ticketing workflow

A field-ops product usually has at least two speeds of work:

- **Ticketing workflow** — something broke in the world; a human must
  be assigned, dispatched, and the visit closed. Latency and
  availability of *assignment* matter. The data is operational and
  contentious (this technician, this slot, this site).
- **Nonticketing workflow** — invoicing, parts inventory, knowledge
  articles, contracts, maybe customer master data. Different cadence,
  different users, often heavier reporting. Still attached to the same
  customers and sites.

They look like one product because the customer is one company. They
are not one change-rate, one scale curve, or one failure domain. That
is why this case is a teaching tool: a single monolith is plausible,
and splitting it is never free.

### A bad scenario

Imagine one deployable, one schema, one release train. Finance needs a
tax-table change in invoicing. Dispatch needs a bugfix so Saturday
on-call assignment does not double-book. The tax change takes a lock
on a table that assignment also reads. The release is all-or-nothing.
A rollback of billing rolls back the dispatch fix. Nobody can say,
from the outside, which "component" failed — the process is one.

That is the scenario this track keeps returning to. It is not
"monoliths are evil." It is: **coupled release + coupled data +
coupled failure** with two workflows that did not want any of those
couplings. Name that pain before you name a target architecture.

### Components

Before services, name **components**: clumps of code that do one job a
domain expert would recognize. Intake, assignment, dispatch, invoicing,
inventory, notifications, identity. Not classes. Not "the API layer."
Not "utils."

Chapter 1 only needs you to see that the running case *has* parts,
even while they compile together. [Chapter 5](../5-component-patterns/)
is the method for sizing and grouping them. If you cannot name the
parts, you cannot later claim you decomposed anything.

### Data model

Sketch only, on purpose. You need enough nouns to see the traps:

```
  customer ----< ticket >---- technician
      |             |
      |             +---- work_order ----< part_line
      |
      +----< invoice >---- tax_line
```

Shared keys (`customer`, `site`, `technician`) are the seams that will
tempt a shared database. Tickets and invoices look joinable because
they are, in one company. That join is the future distributed
transaction. Do not "solve" it in this chapter. Just refuse to pretend
the boxes on the whiteboard are independent while they share this
graph.

When you take notes on your own system, draw this much: the nouns that
two workflows both think they own, and the foreign keys you would have
to break to deploy those workflows apart. That drawing is the input to
the rest of the track.

## Check yourself

1. A colleague says "event-driven is the best practice for this." What
   two questions do you ask before agreeing? Name the failure if you
   skip the questions.
2. Pick a decision from a system you have shipped that was expensive to
   reverse. Was it architecture or design by this chapter's test? What
   would an ADR have needed to record?
3. Why does "it depends" fail as advice until you name the dependence?
   Give a production example where two teams used the same pattern and
   only one was right.
4. You have twelve microservices and one Postgres. How many
   *architectural* boundaries do you actually have, and which later
   chapter is going to make that precise?
5. Write the outline of an ADR (title, context, decision, one good
   consequence, one bad) for "we will keep ticketing and invoicing on
   the same schema for now." What fitness function would you attach?
6. A fitness function lives in a wiki and never fails a build. What
   failure mode is that, and what is the smallest CI check you would
   add this week on a real repo you know?
7. Where does data show up as architecture in a product you know —
   a table, a key, a derived index — and what went wrong when someone
   treated it as an implementation detail?
8. Why does this track want one running case instead of a fresh domain
   per chapter? What goes missing in your notes if you skip that
   discipline?
9. Ticketing vs nonticketing: name one characteristic each workflow
   would optimize, and one coupling you would *not* want between them.
   What incident does that coupling cause if you leave them in one
   process?
10. DDIA chapter 1 also talks about trade-offs. In one sentence, what
    job is *this* chapter doing that that one is not?

Continue to [Discerning coupling](../2-coupling/).

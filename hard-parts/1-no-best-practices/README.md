# What happens when there are no "best practices"?

Software Architecture: The Hard Parts, Ford, Richards, Sadalage, Dehghani, O'Reilly 2021, Chapter 1.

## What the subject is

Architecture is the set of decisions that are expensive to reverse. Distributed design has no catalog of recipes that fit every system. The method is to name the trade-off, write down why this side was picked, and attach a check that fails when that reason goes stale.

The easy parts already have recipes: how to terminate TLS, how to take a database backup, how to put a health check on a process. The hard parts are the questions where every option is wrong for someone in the room.

- Split this monolith now, or pay the coordination tax for another year.
- Let two workflows share a table, or copy the data and accept lag.
- Make a synchronous call, which is simpler to read, or publish an event and couple two deploy schedules.

Those are structural bets. A granularity choice is also a data-ownership choice, and it is also the shape of any saga that has to cross the cut. Another company's service cut cannot be pasted onto this system.

A practice fits a situation: team skill, data gravity, regulatory blast radius, and how often the bit changes. There is no best practice with the situation removed. Characteristics fight. Independent deploy fights a single ACID transaction. Elastic scale fights a chatty in-process call. Write the conflict down. Decisions that are irreversible, or only miserable to reverse, get more ceremony than reversible ones. Database identity, public contracts, and who owns a row outlive class names. "It depends" is a finished sentence when the dependence is named. An architecture decision record captures that dependence. A fitness function keeps the record from rotting.

"We always use Kafka," "we always use Kubernetes," or "one service per table" is an unevaluated habit. A decision with no record and no check gets relitigated every quarter, or it gets treated as physics by people who were not in the room.

## One field-ops product

Hang every decision on one system. Switching products every time hides the path from a coupling choice to the saga that coupling forces. The nouns never sit on the same page.

Use a field-ops product with two speeds of work.

- **Ticketing.** Something broke. A human must be assigned, dispatched, and the visit closed. Latency and availability of assignment matter. The data is operational and contentious: this technician, this slot, this site.
- **Nonticketing.** Invoicing, parts inventory, knowledge articles, contracts, and maybe customer master data. Different cadence, different users, often heavier reporting. The same customers and sites are still attached.

They look like one product because one company is the customer. They are not one change-rate, one scale curve, or one failure domain. A single monolith is a plausible start. Splitting it has a price.

The bad picture is one deployable, one schema, and one release train. Finance needs a tax-table change in invoicing. Dispatch needs a bugfix so Saturday on-call assignment does not double-book. The tax change takes a lock on a table that assignment also reads. The release is all-or-nothing. A rollback of billing rolls back the dispatch fix. From the outside, nobody can name which component failed, because the process is one. Name that pain before naming a target architecture: coupled release, coupled data, and coupled failure, on two workflows that did not want those couplings.

Name **components** before services. A component is a clump of code that does one job a domain expert would recognize: intake, assignment, dispatch, invoicing, inventory, notifications, identity. A class, "the API layer," and "utils" are the wrong grain. The parts exist even while they compile together. A claim that the system was decomposed needs those names. Naming and grouping components is a method for finding architecture. Whether a function is recursive is design, inside a boundary.

This much of the data model is enough:

```
  customer ----< ticket >---- technician
      |             |
      |             +---- work_order ----< part_line
      |
      +----< invoice >---- tax_line
```

Shared keys (`customer`, `site`, `technician`) are the seams that tempt a shared database. Tickets and invoices are joinable because they belong to one company. That join is the future distributed transaction. The boxes on a whiteboard stay dependent while they share this graph. On any other system, draw the same two things: the nouns both workflows think they own, and the foreign keys that would have to break before those workflows can deploy apart.

## Write the decision down

An **architecture decision record** (ADR) is a short, dated note of one architecture-significant choice: the context you had, what you picked, and the downside you accepted. It records a decision. A design spec, a wiki novel, and a ticket comment that disappears in the next tracker migration do something else.

Write an ADR when reversing the choice would take weeks or would strand data. A library version, a class rename, and a local refactor that does not move a boundary do not need one. Ceremony on every pull request trains people to skip the ceremony that matters.

Keep the shape boring:

1. **Title.** The decision, in one sentence. "Billing and dispatch stay on one Postgres until we have a customer identifier we can replicate."
2. **Status.** Proposed, accepted, or superseded. An ADR dies in public. It is not edited into a different past.
3. **Context.** Forces, constraints, and the options considered. Enough that a new hire can see why the choice was hard.
4. **Decision.** What you will do. One pick. A menu is unfinished.
5. **Consequences.** Good and bad. The bad ones are the point. A trade-off with no named downside is unfinished.

Keep the files in the repo, next to the code they constrain, numbered, and grepable. Review the ADR in the same change that implements the decision. When the context changes, supersede the record with a new ADR. Leave the old text as history.

On the release train, the title above is a real pick. Context: invoicing and assignment share a schema, a tax-table lock blocks Saturday assignment, the two groups want different calendars, and no customer identifier is stable enough to replicate. The good consequence is one ACID transaction, with no lag and no reconciliation job. The bad consequence is that lockstep releases continue, and a tax migration can still lock assignment. A different pick, with the loser named, is "we accept slower billing deploys so that dispatch can ship daily." Dispatch wins cadence. Billing pays. "Microservices" names neither side.

A payments team records a second kind of pick: "we will dual-write to the ledger and the card processor, and treat the ledger as source of truth on mismatch." Eighteen months later a new orchestrator is proposed. The ADR says whether that move is allowed, and which fitness check has to move with it.

The people who chose a shared database leave. With the ADR, the next team can see which reports, locks, and night jobs depend on the join. Without it, they treat the sharing as a law of nature, or they rip it out blind.

## Attach a check that can fail

A **fitness function** is an automated, or at least scheduled, check that an architectural characteristic still holds. It is a unit test for a property of the system, not for a method. The idea comes from evolutionary architecture and is used here as a working tool. You have modularity when a check fails because someone imported across a forbidden boundary, or because the p99 of assign-technician crossed the line written in the ADR. A slide does not establish the property.

For each characteristic an ADR claims, attach something that can go red.

- **Static / structural.** ArchUnit, import-linter, dependency-cruiser, or a CI job that fails on a cycle between `billing/` and `dispatch/`. Cheap. It catches the leak where someone imports a helper across the boundary.
- **Runtime / operational.** A synthetic probe or an SLO burn for "dispatch p99 < 200 ms." A chaos experiment in which billing is down and intake stays up. A contract test against a published schema.
- **Pipeline gates and continual checks.** Some checks run on every merge: cycles, forbidden tables. Some run in production all week: error budget, queue lag. A nightly dashboard is a different tool from a merge gate.

A fitness function sits on the delivery path. A wiki page of intended checks is a wish. A red build is a fitness function.

The usual categories — atomic versus holistic, triggered versus continual — are a coverage lens. They show that a team has only static checks. They are not a card to fill in.

Start with less machinery. One cycle check, and one rule that this module must not touch that table, teaches more than a framework with forty empty hooks. Add a check when an ADR claims a characteristic.

After a reporting query locks `work_orders`, the fitness function bans that join from the OLTP connection role. The next dashboard author hits a red pipeline. The lesson arrives before the Saturday outage.

## Separate architecture from design

Use two definitions in a review.

- **Architecture.** Decisions that are expensive to reverse: boundaries, communication style, data ownership, the shape of deployables, and the characteristics you are optimizing (deployability, scale, availability). Changing your mind means migrating data, versioning a contract, or coordinating three teams.
- **Design.** Decisions inside a boundary that one team can redo without the rest of the company: class structure, local caching, which helper library, how a module names its functions.

The line moves. A small interface becomes architecture the moment two other quanta depend on it. A **quantum** is the smallest set of code and data that can be deployed independently and still hangs together. A large diagram that only one team will ever implement is design with extra boxes.

The test is the cost of being wrong for a year. "Rename and move on" is design. "We run two systems of record and a reconciliation job" is architecture: write the ADR and attach the fitness function.

On this product, whether `ticket` and `invoice` share a schema is architecture. Renaming a helper inside assignment is design. The failure pattern is architects spending the review on class hierarchies while engineers make an irreversible data cut in a pull request with no ADR.

## Data is the part that stays

Code moves. Data sits. A function can be copied into a new service in an afternoon. A table that three workflows join will still be there after the reorg, the rewrite, and the temporary dual-write.

When two services must still release on the same Friday, look at the schema. Shared rows, shared sequences, shared status enums, and foreign keys that cross a hoped-for boundary are architecture. They are the thing that makes a service cut real or fake.

Three consequences are immediate.

- Two teams that cannot change a column without a meeting do not have independent architecture, whatever the repository count says.
- Analytical copies, search indexes, and caches belong to the architecture even when someone calls them derived. They constrain deletion, privacy, and what "done" means for a write.
- A data model that was convenient inside one deployable becomes a distributed transaction the moment that process is cut in half.

## What goes wrong

A design review asks for the industry best practice. The room picks a pattern because it is popular. Six months later that pattern is the incident. The pick needed a named winner and a named loser.

Collecting practices, and skipping consequences, produces a diagram that looks current and coupling that matches last year.

ADRs fail as bureaucracy: templates with twenty empty headings, or a graveyard of pages nobody reads. They also fail by absence, so every incident starts with archaeology. The failure is a decision that cannot be found when the trade-off turns. The template is incidental.

Fitness functions fail as theater: flaky checks people mute, metrics nobody owns, or a gate so strict that teams route around it with a shared utils jar. They also fail by absence. With no checks, the architecture drawn on day one is fiction by month six. A check that lives in a wiki and never fails a build is the wish.

Architecture as an ivory tower is structure with no delivery. Design as accidental architecture means every merge can add a cross-service join. Both leave a system that cannot be changed on purpose.

## Recap

Name the trade-off on one product: who wins, who pays, and which situation the pick depends on. Record it as an ADR with context, one decision, and a downside, stored next to the code and superseded in public when the context changes. Attach a fitness function that can go red on the merge or in production. Treat shared rows, keys, sequences, status enums, and derived copies as architecture. Call the choice architecture when a year of being wrong means a data migration, a versioned contract, or a reconciliation job.

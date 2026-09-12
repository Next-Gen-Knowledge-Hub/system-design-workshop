# 2. Discerning coupling

Companion notes for **Chapter 2** of *Software Architecture: The Hard Parts*
(Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani; O'Reilly,
October 2021).

If chapter 1 said "name the trade-off," this chapter names the thing
the trade-off is usually about: **coupling**. Skip it and you will count
Git repos, Kubernetes deployments, or "bounded contexts" on a slide and
think you have a distributed system. Then a schema change, a shared
library bump, or a synchronous call on the checkout path will prove you
still had *one* thing that had to move together. The rest of this track
— granularity, ownership, sagas — is unusable until you can see the
real unit of architecture.

**See also (do not merge):** [DDIA ch. 9](../../ddia/9-distributed-trouble/)
is why networks lie (timeouts, clocks, partial failure). This folder is
how to *see* whether two boxes are actually one deployable unit. Partial
failure is what dynamic coupling feels like; it is not the definition
of a quantum.

## The mental model

```
  slide says "two services"
       |
       v
  can you deploy A on Tuesday
  without shipping B, and without
  migrating a table B owns?          ---- no ----+
       |                                         |
      yes                                        v
       |                              +--------------------+
       v                              |   ONE QUANTUM      |
  +-------------------------+         |  A + B + shared DB |
  | two quanta              |         |  (two processes,   |
  |  [A + A's data]         |         |   one architecture)|
  |       |  runtime call   |         +--------------------+
  |  [B + B's data]         |
  +-------------------------+
```

The one sentence to remember a year from now: **an architecture quantum
is the smallest thing you can deploy independently that still hangs
together** — cohesive in purpose, glued by static coupling on the
inside, talking to other quanta only through dynamic coupling you can
name.

Two consequences fall straight out of that diagram. First, a
microservice that cannot ship without another team's database
migration is not a second quantum; it is a second process. Second,
"loosely coupled" is not a feeling. You have to say whether you mean
*static* (source, schema, contract, shared lib) or *dynamic* (runtime
call, message, temporal need to be up).

## Architecture quantum

The **architecture quantum** (plural **quanta**) is the unit this book
uses instead of "service," "module," or "bounded context," because those
words get used for wishful diagrams. A quantum has four properties you
can check. Miss one, and you are describing a hope.

### Independently deployable

You can release this artifact through production without releasing the
others *as a coordinated set*. Same-day coincidence is allowed;
*required* lockstep is not.

**Problem** — Release trains: billing, dispatch, and the shared "platform"
jar always go out together because a column default or a DTO field
ships in all three.

**Solution** — Ask the last six production releases: which artifacts
*had* to move in one change request? That set is the quantum, even if
the org chart shows three teams. Independence is an operational fact,
not a repo naming convention.

**Failure mode** — You split the build and keep a single migration
folder. Deployables multiply; deploy *independence* does not. Friday
still needs a war room.

Independence is also about rollback. If you cannot roll A back without
rolling B back, you did not decouple the release; you synchronized two
failure stories.

### High functional cohesion

The code and data inside the quantum exist to do **one** business
capability a domain expert would name in one breath: "assign a
technician," "issue an invoice," "reserve a part." Cohesion is not
"these classes share a utility package." It is "this stuff changes for
the same reason."

**Problem** — A "customer service" that also renders PDFs, prices tax,
and pages on-call, because all of those screens sit in one app.

**Solution** — Group by change reason and by the workflow in the
[book's ticketing/field-ops case](../1-no-best-practices/) (the label,
not the plot): ticketing assignment vs invoicing are different reasons
to ship, even when both mention `customer_id`.

**Failure mode** — Accidental cohesion: everything that touches HTTP,
or everything that touches `Customer`. You cannot scale, test, or
replace a capability because it was never a capability — it was a
layer.

Cohesion is what makes a quantum *worth* deploying independently. A
perfectly independent 40-line "true/false" service is independent and
useless. [Chapter 7](../7-service-granularity/) will push on that
other edge. Here, just refuse to call a technical layer a quantum.

### High static coupling

**Static coupling** is the coupling you can see without running the
system: source imports, compile-time references, a shared database
schema, a shared library version, a contract that must change in lockstep,
a foreign key you cannot migrate alone.

Inside a quantum, static coupling should be **high**. That is not a
bug. The classes, tables, and internal APIs *should* know each other;
that is cohesion with teeth. The bug is high static coupling *across*
things you are calling separate quanta.

```
  inside one quantum (wanted):
    assign.py --> slot.py --> work_orders table
    (ship together; one schema; one rollback)

  across quanta (unwanted static):
    assign.py --> invoices table
    assign.jar --> common-1.8.jar <-- billing.jar
    (you think you have two services; you have one quantum)
```

**Problem** — Teams "decouple" by putting an HTTP boundary in front of
the same tables and the same shared kernel.

**Solution** — Inventory static couplings the way you would inventory
secrets. For each pair of hoped-for services, list: shared DB objects,
shared libraries that carry domain types, generated stubs that break
when either side moves, foreign keys, and "common" packages that both
compile against. If the list is not empty, you do not have two quanta
yet. You have a distributed monolith, or you have *one* quantum with a
network hop.

**Failure mode** — Measuring coupling only as "number of REST calls."
A system with few calls and one schema is still one quantum. A system
with many async messages and separate data stores may be several.

Static coupling is also where data gravity from
[ch. 1](../1-no-best-practices/) shows up as a measurable thing. You
will break it on purpose in [ch. 6](../6-operational-data/). This
chapter only trains your eye.

### Dynamic quantum coupling

**Dynamic coupling** is what happens at runtime between quanta: a
synchronous call, an asynchronous message, a workflow that waits, a
cache fill, a "must be up or I cannot start." You cannot eliminate it
in a useful distributed system. You *can* name its shape, because the
shape is the operational contract.

Useful distinctions, kept practical:

- **Synchronous vs asynchronous.** Sync: the caller is stuck until the
  callee answers or times out. You have just imported the callee's
  availability into the caller's SLO. Async: you have imported *lag*
  and "what if the message never comes" instead.
- **Temporal.** Even async coupling is temporal if the workflow cannot
  proceed until the other side processes. A queue does not magically
  remove coupling; it changes the failure mode from HTTP 503 to "stuck
  ticket."
- **Data vs control.** Passing a document you already own is different
  from asking another quantum to *decide*. Control-style coupling
  (orchestration callbacks, "is this technician allowed?") is how
  two quanta become one business transaction in disguise.

**Problem** — "We're loosely coupled; we use events." The consumer
still cannot deploy until the producer's new required field ships, and
the UI still waits on the consumer's handler.

**Solution** — Draw the runtime arrow and label it: sync/async, who
waits, who owns the data in the payload, what happens when the other
side is gone for an hour. That drawing *is* dynamic quantum coupling.
[Chapter 11](../11-distributed-workflows/) and
[ch. 13](../13-contracts/) will refine the arrow. Here you only need
to stop calling it "loose" without a label.

**Failure mode** — Treating dynamic coupling as the only coupling.
You rewrite REST to Kafka and keep the shared `customers` table. Static
coupling remains; you added an ops surface.

Dynamic coupling is allowed between quanta. It is the *point* of having
more than one. The discipline is: no *hidden* static coupling riding
along with the call.

## Understanding quanta

The book's ticketing/field-ops case is a running example of discovering
that the number of quanta is smaller than the number of boxes. Use the
label; do the discovery on a system you actually operate. The method is
the same whether you are looking at a monolith, a "modular" monolith, or
a fleet of services.

### Count processes last

Start from data and release history, not from `docker compose`.

1. **List deployables** you *think* are independent.
2. **Cluster by static coupling.** Shared schema, shared domain library,
   lockstep contract, must-ship-together migrations. Each cluster is a
   candidate quantum.
3. **Check cohesion.** If a cluster contains two change-reasons (tax
   tables and on-call assignment), you may have *under-split* inside
   the quantum — a later decomposition problem, not a reason to pretend
   they are already two quanta.
4. **Label the remaining arrows** as dynamic coupling. Those arrows are
   the architecture you actually have between quanta.

```
  hoped-for map:     [intake] [assign] [invoice] [parts]
                          4 "microservices"

  static reality:    [intake+assign+shared OLTP]
                     [invoice+parts+same OLTP]     still 1 if same DB
                     --------------------------------------------
                     one quantum until the database splits
```

A modular monolith can still be **one** quantum: independently
deployable as a unit, internally cohesive-ish, highly statically
coupled, with no other quanta to talk to. That is a valid architecture.
It becomes a lie only when you *claim* independent deploy of the
modules and cannot do it.

A "service per class" fleet can still be **one** quantum if they share
a kernel and a database. Quanta are not a reward for YAML.

### What independent does not mean

- **Not** "runs in its own process." Lots of processes, one quantum.
- **Not** "owned by one team." Team topology should *follow* quanta;
  it does not create them. Two teams on one quantum is a coordination
  tax. One team on five accidental quanta is paging tax.
- **Not** "has an API." Every class can have an API. The question is
  whether the other side can survive a Tuesday when you do not ship.
- **Not** "async somewhere." See dynamic coupling above.

Independent *does* mean: different release cadence is possible, rollback
is possible, and a fatal GC in invoicing does not have to kill
assignment — unless you reintroduced that death via a sync call on the
hot path (dynamic coupling eating the availability you paid for).

### Quanta in the running case

Apply the four properties to the ticketing vs nonticketing split from
[ch. 1](../1-no-best-practices/) without needing any of the book's
story beats:

- If assignment and invoicing share `customers` and `work_orders` in
  one schema, **static coupling is high across the hoped-for cut**.
  One quantum.
- If you extract invoicing to its own deployable but assignment calls
  it synchronously on every close-ticket, you bought a process boundary
  and donated your availability. Two candidate quanta with tight
  **dynamic** coupling — maybe still the wrong cut, maybe a temporary
  step. Name it.
- If invoicing consumes `TicketClosed` and owns `invoices`, and the two
  can ship on different days, you are approaching two quanta. You now
  have lag and a reconciliation story. That is the trade-off, not a
  failure of nerve.

Do this on *your* system in a design review: "How many quanta, and what
evidence?" Anyone who answers with a count of repositories has not
done chapter 2.

### Why this chapter sits before modularity

[Chapter 3](../3-modularity/) will argue *why* you might want more
quanta (maintain, test, deploy, scale, survive). If you cannot see
quanta, that argument becomes "why microservices are good," which is
how you oversplit. Coupling first, drivers second, decomposition
method third. The order is the point.

## Check yourself

1. Take a pair of "services" you have actually shipped. Are they one
   quantum or two? Which of the four properties failed, and what
   incident was the giveaway?
2. Why is high static coupling *inside* a quantum desirable? What
   failure mode appears if you try to drive that static coupling to
   zero inside one capability?
3. A team replaces REST with a message bus and keeps one database.
   What kind of coupling did they change, and what kind did they not?
   Give a production-shaped failure that remains.
4. Independently deployable vs independently *rollbackable*: describe a
   case where you could ship A without B but could not undo A without
   B. Is that one quantum?
5. Functional cohesion: split "everything that touches Customer" from
   "everything that changes for the same business reason." Which one
   yields a quantum, and what goes wrong in a real codebase if you pick
   the first?
6. Draw dynamic coupling for a checkout (or ticket-close) path you
   know. Label each arrow sync/async and say whose SLO just absorbed
   whose. Where would an hour-long outage of the callee leave the
   caller?
7. How can a modular monolith be a single honest quantum, and when
   does calling it "modular" become a lie? Use release evidence, not
   package names.
8. Someone counts bounded contexts on a domain diagram and calls that
   the number of quanta. What measurement would you demand instead?
9. In the book's ticketing/field-ops case (as a label only), why would
   sharing `work_orders` across assignment and invoicing collapse two
   hoped-for services into one quantum? What later chapter attacks
   that sharing?
10. Name a fitness function ([ch. 1](../1-no-best-practices/)) you
    could automate to detect a *new* static coupling between two
    claimed quanta. What failure do you prevent on the next "tiny
    helper" import?

Continue to [Architectural modularity](../3-modularity/).

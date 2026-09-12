# 12. Transactional sagas

Companion notes for **Chapter 12** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

[Chapter 11](../11-distributed-workflows/) picked a **style**
(conductor vs reactions) and a **state owner**. This chapter is what
you do when the plot must **survive a partial failure**: service B
committed, service C timed out, the customer is staring at a spinner.
A saga is the application-level protocol for that. It is not a
database feature.

## See also (do not merge)

**Isolation, write skew, serializability, two-phase commit** — protocols
a *store* runs so concurrent sessions and crashes do not smash rows —
are [DDIA ch. 8](../../ddia/8-transactions/). A saga is an
**application protocol**. Isolation will not compensate a charge in
Billing because Dispatch ran out of techs. 2PC will not save you
across Billing's Postgres and Dispatch's document store unless you
actually installed XA and accepted blocking prepares. Keep the
folders apart: one is "what the DB promises," the other is "what our
code promises when the DB cannot."

## The mental model

Do not memorize eight fairy tales. Sit them on a cube. Three axes,
eight corners, cute names as handles.

```
           orchestrated (o)              choreographed (c)
         +----------------------------+----------------------------+
 atomic  | seq: Epic        (sao)     | seq: Phone Tag    (sac)    |
  (a)    | par: Fantasy Fiction (aao) | par: Horror Story (aac)    |
         +----------------------------+----------------------------+
 event.  | seq: Fairy Tale  (seo)     | seq: Time Travel  (sec)    |
  (e)    | par: Parallel    (aeo)     | par: Anthology    (aec)    |
         +----------------------------+----------------------------+

  seq (s) = one step waits for the previous
  par     = several steps in flight  (book's first-letter "a":
            asynchronous / parallel — NOT the atomic "a")
```

The codes overload the letter **a**. First position: sequential vs
parallel. Second: atomic vs eventual. Third: orchestrated vs
choreographed. If you find yourself reciting "SAO means Epic," you
are studying the wrong object. Study the **cell**.

The one sentence to remember: **a saga is a state machine plus an
undo story**; the cube only says who drives the machine, whether
strangers see intermediate writes, and whether steps queue or fan
out.

Two consequences. First, "we use the saga pattern" is as empty as
"we use transactions." Name the cell, or name the three axes. Second,
the atomic half of the cube often collapses into **distributed
commit** (2PC and friends). That is a legal cell. It is usually the
cell you were trying to leave when you split the database
([ch. 9](../9-data-ownership/)).

## Three axes, not a catalog

### Atomic vs eventual (a vs e)

**Problem** — Product language says "the booking either happens or
it doesn't." After the split, that sentence is no longer free.

**Solution** — Decide whether **intermediate writes are allowed to
be visible**.

- **Atomic** here means: the plot tries to look like one commit.
  Participants prepare or hold; outsiders should not see "charged
  but not assigned." This is the 2PC instinct, or a long-lived lock,
  or a reservation that is not quite a sale until the end.
- **Eventual** here means: each participant **commits locally** when
  it is asked. The world *will* see "charged, not yet assigned."
  If a later step fails, you run a **compensating update** — a new
  business operation that semantically undoes (refund, release
  tech, cancel hold). You do not un-write the WAL.

**Failure** — Mixing the words. "Eventual consistency" in
[DDIA replication](../../ddia/6-replication/) is lag between copies
of the *same* data. "Eventual" on this axis is **cross-service
business state** that is allowed to be half-done on purpose.
Calling a saga "ACID" because the first letter of the acronym is A
is how you skip the undo story.

Atomic sagas buy a simpler customer story and cost **availability
and holding time**: participants must be up, and they hold
resources until the plot decides. Eventual sagas buy survival of
partial failure and cost **visible inconsistency plus compensations
that can themselves fail**.

### Orchestrated vs choreographed (o vs c)

**Problem** — Style in [ch. 11](../11-distributed-workflows/) was
the happy plot. Failure splits the plot in two: remaining dos and
the undos. If those have different owners, you designed two systems.

**Solution** — Who decides the next step *and* who decides to undo
should share a style, or the hybrid must be drawn on purpose.

- **Orchestrated:** the conductor's state machine fires commands
  (`do`, `undo`). One place to timeout. One place to see "compensating."
- **Choreographed:** failure is an event (`ChargeFailed`,
  `NeedToReleaseTech`). Whoever subscribed to the happy path must
  have a twin for the sad path, or the undo never runs.

**Failure** — Happy-path choreography with orchestrated undo (or
the reverse) that nobody drew. Compensations that only exist in
the conductor while the happy path lives in listeners: two plots,
two bugs.

### Sequential vs parallel (the remaining letter)

**Problem** — A plot with independent steps (fraud score *and*
inventory hold) should not wait in a single file. A plot with
*dependent* steps (charge **then** dispatch) must.

**Solution** — **Sequential (`s`)** means step *n+1* does not start
until step *n* finishes (commit or prepare). **Parallel** (first-
letter `a` in the book's codes) means the conductor or the event
graph **fans out**: several participants in flight, then a join.

Parallel buys latency. It costs a harder machine: you must record
*which* of N succeeded when you abort. Sequential buys a trivial
undo order (reverse the tape). It costs tail latency: the slowest
dependent step gates the rest.

**Failure** — Fan-out without a join. Three parallel steps, two
succeed, nobody owns "we are 2/3 done and the third is dead."
That is not parallelism. That is a leak.

## What each cell buys and costs

Walk the cube by **region**, not by nickname. The names are
mnemonics for slides. The costs are the architecture.

### Atomic + sequential

**Epic (sao)** — A conductor walks participants one by one and
tries to keep the plot **uncommitted to the outside** until the
end. Think: directed 2PC, or a booking that holds every row until
the last yes.

- Buys: one story, one timeout surface, undo is "abort" more than
  "refund."
- Costs: the conductor is a hub; holds and prepares span the *sum*
  of step times; a down participant blocks the plot; you are in
  [DDIA 8](../../ddia/8-transactions/) operationally (in-doubt,
  coordinator recovery) even if you never said XA.

**Phone Tag (sac)** — Same atomic sequential ambition, **no
conductor**. The "transaction context" is passed from service to
service like a token: A prepares, calls B, B prepares, calls C.
The last one says commit, or someone says abort, and the token
walks back.

- Buys: no hub process.
- Costs: the token *is* the hub; it is worse, because it is
  **moving**. Who holds the locks if B dies with the token? Who
  Support-calls? Debugging is literally phone tag. Atomicity
  without a coordinator is a research problem you should not
  re-solve in a ticket app.

### Atomic + parallel

**Fantasy Fiction (aao)** — A conductor **prepares several
participants at once**, then commits or aborts all. Parallel 2PC
with a boss.

- Buys: lower latency than Epic if steps are independent; still a
  single "happened or not."
- Costs: every 2PC pain **plus** a join. The fantasy is "we can
  have all-or-nothing *and* fan-out *and* independent deploys."
  You can have two. The third leaks as blocked prepares and a
  god orchestrator.

**Horror Story (aac)** — Parallel atomic **without** a boss.
Several services try to agree they all prepared, via events.

- Buys: theoretically no hub and no visible half-state.
- Costs: distributed deadlock, unclear who decides, timeout
  storms, "are we committed?" as a rumor. The name is the review
  comment. If you need this cell, you probably need a real
  atomic-commit protocol with a named coordinator — which means
  you left this cell for Fantasy Fiction or for **one database**.

If a design review lands in the atomic half, stop and ask: should
these writes be **one architecture quantum** again
([ch. 2](../2-coupling/), [ch. 7](../7-service-granularity/))?
Merging is a valid saga strategy. It is the one with the fewest
letters.

### Eventual + sequential

**Fairy Tale (seo)** — The workhorse. Conductor tells A: commit
your local work. Then B. Then C. On failure at C, conductor tells
B: compensate, then A: compensate. State machine is boring and
**central**.

- Buys: no distributed prepare; participants stay up only for
  their own step; one place for timeout/retry/compensate; you can
  test the machine.
- Costs: hub coupling ([ch. 11](../11-distributed-workflows/));
  customers see intermediate states; you must **write** undos that
  are real operations, not DB rollbacks; the fairy tale ends when
  a compensation fails (money you cannot refund automatically).

**Time Travel (sec)** — Sequential eventual, **events**. A commits
and publishes. B reacts, commits, publishes. C fails. A
**rewinding** chain of compensations is supposed to walk back
through time: `CFailed` → B undoes → `BUndone` → A undoes.

- Buys: no conductor; participants deploy independently on the
  happy path.
- Costs: the rewind is a second choreography you will forget.
  Event order, late duplicates, and "who is allowed to emit undo"
  become the product. Observability is time travel: you reconstruct
  the plot from a log after the customer already called. This cell
  is where [DDIA 12](../../ddia/12-stream-processing/) pipes get
  mistaken for a saga runtime. A log can *record* the rewind. It
  will not *design* it.

### Eventual + parallel

**Parallel (aeo)** — Conductor fans out to independent
participants, each commits locally, conductor **joins**. Any
failure: compensate the ones that already committed, skip the
ones that did not.

- Buys: latency; still one state row that lists N children;
  timeout is still a conductor feature.
- Costs: join logic; partial success is the **common** case, not
  the edge; compensating a subset without double-undo; the
  conductor's machine is a product (pending / 2-of-3 / compensating
  / done).

**Anthology (aec)** — Parallel eventual choreography. Several
independent event-stories that together you *call* a saga. Each
short story commits locally. "Done" is a distributed opinion.

- Buys: maximum deploy independence; no hub; steps that truly do
  not care about each other do not wait.
- Costs: no single picture; compensation is N independent undos
  with no boss to notice a missing one; "are we done?" wants a
  join you refused to build. Anthologies need an explicit
  **completion rule** (and often a tracker — hello, hub). Without
  that rule you have concurrent activity, not a saga.

## State machines and eventual consistency

**Problem** — Eventual cells make half-done **normal**. If you
only model happy paths, Support is your state machine.

**Solution** — Write the machine. Orchestrated: one row per
instance, explicit states, explicit transitions.

```
  accepted -> (step i running) -> step i done -> ...
           -> compensating -> compensated
           -> failed (parked / human)
           -> complete
```

For parallel eventual, the "step i" box is a **set**: children
`{B: done, C: running, D: failed}`. The join is a transition, not
a hope.

Choreographed: each participant has a **local** machine
(`received`, `done`, `undoing`, `undone`). The global machine is
either a projection you built or a story you tell in incident
docs. If you need to query the global machine, you have admitted
a state owner. Build it.

Eventual consistency on this axis is **application-visible**.
UI copy, idempotent retries, and "your refund is processing" are
part of the protocol. Hiding half-state with a spinner that sits
until the saga completes is secretly trying to climb back to
atomic without holding prepares. Sometimes that is the right UX.
Name it: you are **blocking the user**, not the databases.

**Failure** — Status enums that only include `SUCCESS` and
`ERROR`. Missing `COMPENSATING` is how you double-refund when a
retry hits a step that already undid. Missing deadlines is how
sagas live forever in `RUNNING`.

## Techniques: compensations and the rest

**Problem** — "We'll just roll back" is not available. The write
already committed in someone else's database.

**Solution** — Compensating updates are **new** commands with
business meaning:

- Charge → refund (not "delete the ledger row").
- Reserve tech → release reservation.
- Send email → send correction (you cannot unsay; you **pivot**).

Rules that keep this from becoming folklore:

1. **Idempotency** on do *and* undo. Keys: `saga_id + step +
   direction`. Retries will happen ([DDIA 9](../../ddia/9-distributed-trouble/)
   timeouts do not mean "it didn't commit").
2. **Reservations / semantic locks** when eventual visibility is
   too loud: hold inventory as `HELD`, convert to `SOLD` or
   `RELEASED`. That is a small atomic story *inside* one owner,
   wrapping an eventual saga outside.
3. **Forward recovery** beats compensate when completing is
   cheaper than undoing (retry Dispatch; do not refund a charge
   you can still fulfill).
4. **Timeouts are transitions**, not logs. When the deadline
   fires, the machine moves.
5. **Park and human** when compensate cannot run (partner API,
   irreversible side effect). A saga that cannot fail-safe must
   fail-loud.
6. **Do not compensate a derived view.** If Search indexed a
   ticket, rebuild from the record
   ([DDIA 13](../../ddia/13-streaming-philosophy/)). Compensations
   are for **other systems of record**.

**Failure** — Compensations that assume they cannot fail.
Compensations that share no id with the original step.
Orchestrators that retry `do` after `undo` already ran. Treating
2PC rollback as a compensation (different protocol, different
visibility).

## How this shows up when you design something

- Name the cell: atomic/eventual, orchestrated/choreographed,
  sequential/parallel. If you cannot, you do not have a saga; you
  have a sequence diagram of the happy path.
- If you picked atomic, say whether you mean **real 2PC** (and
  accept [DDIA 8](../../ddia/8-transactions/) costs) or you mean
  "we will hold reservations." Those are different.
- Write one compensation per local commit. If you cannot, you
  found an irreversible step: pivot, human, or merge the writes.
- Put the state store on the diagram. "Events" is not a store.

## Check yourself

1. In one sentence each: saga vs isolation vs 2PC. Which is an
   application protocol?
2. Why is the first-letter `a` in `aao` not the same word as the
   `a` in atomic? What goes wrong if you conflate them?
3. Epic vs Fairy Tale: same conductor, different visibility.
   What does the customer see on a mid-plot crash in each?
4. Phone Tag vs Time Travel: both choreographed. Which one still
   tries to hide half-writes, and why is that painful without a
   hub?
5. You fan out charge and reserve-tech, then join. Which two
   cells are candidates? What extra state do you store vs
   sequential?
6. Give a step that cannot compensate. What does the machine do
   instead?
7. Why is "Kafka will handle the saga" not a cell on this cube?
8. A refund runs twice. Which technique was missing, and where
   does the key live?
9. When is merging two services a better saga than Anthology?
10. Horror Story: what question has no owner, and what cell do
    you move to if you name one?

Continue to [Contracts](../13-contracts/).

# 12. Transactional sagas

These notes teach sagas — application-level plots that survive a partial failure — from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## What a saga is

A customer opens a ticket. Billing has charged the card. Dispatch times out while reserving a technician. The customer is still looking at a spinner. No single database can roll that back, because the charge and the reservation live in different stores.

A **saga** is the application protocol for that situation. It is a state machine for one run of the plot, plus an undo story for the steps that already committed. It is not a feature you turn on in a database.

Store-level machinery is a different promise. **Isolation** and **serializability** are promises one database makes so concurrent sessions and crashes do not smash its own rows. **Write skew** is one anomaly those promises are about: two transactions read overlapping data, each writes something the other did not see, and the combined result breaks an invariant. **Two-phase commit** (2PC) is a protocol in which participants prepare and then wait for a coordinator to say commit or abort. **XA** is the mechanism that actually runs 2PC across different database products, and it blocks on those prepares. Isolation will not refund a charge because dispatch ran out of technicians. 2PC will not cross billing's Postgres and dispatch's document store unless XA is installed and the blocking prepares are accepted. The database promises what it can inside its own commit. The saga is what the code promises when one commit cannot cover the plot.

An **architecture quantum** is the set of parts that ship and fail together. If a design needs the atomic cells below, the first question is whether those writes should be one quantum again, in one database. Merging is a valid saga strategy. It is the one with the fewest letters.

## Three axes, eight cells

Every saga sits on three axes. Memorizing nicknames without the axes misses the design.

1. **Atomic or eventual.** May outsiders see a half-finished plot?
2. **Orchestrated or choreographed.** Who decides the next step, including the undo?
3. **Sequential or parallel.** Does the next step wait, or are several steps in flight?

The book's three-letter codes overload the letter **a**. First letter: `s` sequential, or `a` parallel (the book also calls this asynchronous). Second letter: `a` atomic, or `e` eventual. Third letter: `o` orchestrated, or `c` choreographed. The `a` in `aao` is parallel. It is not the word "atomic."

| | Orchestrated (`o`) | Choreographed (`c`) |
|---|---|---|
| Atomic (`a`), sequential (`s`) | Epic (`sao`) | Phone Tag (`sac`) |
| Atomic (`a`), parallel (`a`) | Fantasy Fiction (`aao`) | Horror Story (`aac`) |
| Eventual (`e`), sequential (`s`) | Fairy Tale (`seo`) | Time Travel (`sec`) |
| Eventual (`e`), parallel (`a`) | Parallel (`aeo`) | Anthology (`aec`) |

"We use the saga pattern" names none of this. Name the cell, or name the three axes. The atomic half often collapses into distributed commit. That cell is legal. It is also the cell a split database was usually trying to leave.

The worked example below is the same ticket: charge the card, reserve a technician, then send a notice. Fraud scoring and an inventory hold stand in when the steps are independent and can run together.

## Atomic or eventual

Product language says the booking happens or it does not. After the split, that sentence has a price.

**Atomic** means the plot tries to look like one commit. Participants prepare or hold. Outsiders should not see "charged but not assigned." That is the 2PC instinct, or a long-lived lock, or a reservation that is not a sale until the end. Real 2PC — blocking prepares, in-doubt participants, coordinator recovery — and "we will hold reservations" are different designs. Name which one you mean.

**Eventual** means each participant **commits locally** when it is asked. The world will see "charged, not yet assigned." If a later step fails, the saga runs a **compensating update**: a new business operation that undoes the meaning (refund, release the technician, cancel the hold). The database's **write-ahead log** — its durable record of the committed write — stays. The compensation does not un-write it.

On a crash in the middle of the ticket:

- Epic, the atomic orchestrated walk, is still holding or prepared. The customer should not observe a charge without an assignment. The undo is closer to abort than to refund.
- Fairy Tale, the eventual orchestrated walk, has already committed the charge in billing's database. The customer can see it. The undo is a refund, and that refund is itself a business operation that can fail.

Atomic sagas buy a simpler customer story. They cost availability and holding time: participants have to be up, and they hold resources until the plot decides. Eventual sagas survive a participant that commits and then a later participant that fails. They cost a visible half-done state, plus compensations that can fail.

The failure mode is mixing the words. Eventual consistency, as lag between copies of the *same* data, is a replication concern inside one fact. Eventual on this axis is cross-service business state that is allowed to be half-done on purpose. Calling the saga ACID — atomicity, consistency, isolation, durability — because the acronym starts with A, skips the undo story.

## Who drives the do, and who drives the undo

A failure splits the plot into remaining work and undos. If those have different owners, the design is two systems.

**Orchestrated** means the conductor's state machine fires both kinds of command: `do` and `undo`. One place times out. One place can show the status `compensating`.

**Choreographed** means failure is a fact, such as `ChargeFailed` or `NeedToReleaseTech`. Every subscriber on the happy path needs a twin on the sad path. If the twin was never written, the undo never runs.

The failure mode is a hybrid nobody drew. Happy-path listeners, with compensations that exist only inside a conductor, are two plots and two places for bugs. The reverse split has the same shape. Share a style, or draw the hybrid on purpose.

## Sequential or parallel

**Sequential** (`s`) means step *n+1* does not start until step *n* finishes, whether that finish is a local commit or a prepare. Charge, then dispatch, is that order: the later step waits. The undo order is the tape in reverse.

**Parallel** (first letter `a`) means the conductor, or the event graph, fans out. Several participants are in flight, then a join. A fraud score and an inventory hold do not need each other's result, so they can run together. Parallel buys latency. It costs a harder machine: when you abort, you must know which of N already succeeded. Sequential costs tail latency, because the slowest dependent step gates the rest.

The failure mode is fan-out without a join. Three parallel steps, two of them succeeded, and nobody owns "we are 2 of 3, and the third is dead." That is a leak.

## What each cell buys

### Atomic and sequential

**Epic (`sao`).** A conductor walks the participants one by one and keeps the plot uncommitted to the outside until the end. Think of directed 2PC, or a booking that holds every row until the last yes.

The ticket: the conductor tells billing to prepare the charge, then dispatch to prepare the hold, and only then says commit. One story, one timeout surface, and the undo is abort more often than refund. The costs are the hub, holds that last the *sum* of the step times, and a down participant that blocks the whole plot. Operationally this is in-doubt work and coordinator recovery: a participant that prepared and then lost the coordinator does not know whether to commit. That cost arrives even if nobody said "XA."

**Phone Tag (`sac`).** The same atomic sequential ambition, with no conductor process. A transaction context travels as a token. Billing prepares and calls dispatch. Dispatch prepares and calls the notifier. The last participant says commit, or someone says abort, and the token walks back.

There is no hub process. The token *is* the hub, and it is worse because it moves. If dispatch dies holding the token, who still holds the locks, and who does support call? Debugging is phone tag. Atomicity without a coordinator is a research problem; a ticket system should not re-solve it.

### Atomic and parallel

**Fantasy Fiction (`aao`).** A conductor prepares several participants at once, then commits or aborts all of them. Parallel 2PC with a boss.

If the fraud score and the inventory hold are independent, they prepare together, and the customer still sees one outcome. Latency can beat Epic. The cost is every 2PC pain plus a join. The fantasy is all-or-nothing, and fan-out, and independent deploys, all three. Two of those can hold. The third leaks as blocked prepares and a god orchestrator — a conductor that knows every domain and sits on every plot.

**Horror Story (`aac`).** Parallel atomic work with no boss. Several services try to agree, through events, that they have all prepared.

In theory there is no hub and no visible half-state. In practice you get distributed deadlock, no one who decides, timeout storms, and "are we committed?" as a rumor. The question with no owner is that last one. If the design needs this cell, it needs a real atomic-commit protocol with a named coordinator, which means leaving the cell for Fantasy Fiction, or for one database.

### Eventual and sequential

**Fairy Tale (`seo`).** The workhorse. The conductor tells billing to commit the charge locally, then dispatch to commit the reservation, then the notifier to send. If dispatch fails after the charge has committed, the conductor tells billing to compensate.

```
accepted
  -> charge running -> charge done
  -> reserve running -> reserve done
  -> complete

charge done, reserve fails:
  -> compensating (refund)
  -> compensated
  -> failed (parked for a human) if the refund cannot run
```

There is no distributed prepare. Each participant stays up only for its own step. Timeout, retry, and compensate have one home, and the machine can be tested. The costs are hub coupling, customers who see intermediate states, and undos that are real operations. A database rollback is not available for a row that already committed in someone else's store. The fairy tale ends when a compensation fails: money you cannot refund automatically.

**Time Travel (`sec`).** Sequential and eventual, carried by events. Billing commits and publishes. Dispatch reacts, commits, and publishes. Dispatch fails. A rewind is supposed to walk back: `ReserveFailed`, billing undoes, `ChargeUndone`.

Participants deploy independently on the happy path, and there is no conductor process. The rewind is a second choreography, and it is the one teams forget. Event order, late duplicates, and who is allowed to emit the undo become the product. Observability really is time travel: someone reconstructs the plot from a log after the customer has already called. A log can record the rewind. It does not design which undo fires, or in what order.

### Eventual and parallel

**Parallel (`aeo`).** The conductor fans out to independent participants. Each commits locally. The conductor joins. On any failure it compensates the ones that committed and skips the ones that did not.

Charge and reserve-tech, started together, are this cell when half-writes may be visible and a conductor joins them. The other candidate for that fan-out is Fantasy Fiction, if the prepares must stay hidden until the join. Against a sequential saga, the extra state is the set of children: which of N succeeded. One state row can list them, and timeout is still a conductor feature. The costs are the join, partial success as the common case, compensating a subset without undoing twice, and a machine whose product surface is `pending`, `2-of-3`, `compensating`, and `done`.

**Anthology (`aec`).** Parallel eventual choreography. Several independent event-stories, each committing locally, are *called* one saga. "Done" is a distributed opinion.

This buys the most deploy independence. Steps that truly ignore each other do not wait, and there is no hub. Compensation is N independent undos, with no boss to notice a missing one. "Are we done?" wants the join this cell refused to build. An anthology needs an explicit completion rule, and often a tracker — which is a hub again. Without that rule the activity is concurrent work, not a saga. Merging the services beats Anthology when the writes need one outcome, when nobody can name the completion rule, or when the steps are one quantum pretending to be many.

## The state machine is the protocol

Eventual cells make half-done normal. A model that only has a happy path leaves support to be the state machine.

Orchestrated sagas use one row per instance and explicit transitions: `accepted`, step running, step done, `compensating`, `compensated`, `failed` (parked for a human), `complete`. For parallel eventual work, "step i" is a set, such as `{billing: done, dispatch: running, fraud: failed}`. The join is a transition in that row.

Choreographed sagas give each participant a local machine: `received`, `done`, `undoing`, `undone`. The global machine is either a projection you built or a story told in an incident write-up. Needing to query the global machine means you have admitted a state owner. Build that owner.

Eventual, on this axis, is visible to the application. The UI copy, idempotent retries, and the sentence "your refund is processing" are part of the protocol. A spinner that hides the half-state until the saga completes is an attempt to look atomic without holding prepares. Sometimes that is the right user experience. Name what it is: the user is blocked, not the databases.

The failure mode is a status enum with only `SUCCESS` and `ERROR`. Missing `COMPENSATING` is how a retry refunds twice, because it hits a step that already undid. Missing deadlines is how a saga lives forever in `RUNNING`.

## Compensating updates and the moves around them

"Roll back" is not available. The write has committed in another service's database. A compensating update is a **new** command with business meaning.

- Charge becomes refund. Deleting the ledger row is not a refund.
- Reserve a technician becomes release the reservation.
- Send an email becomes send a correction. The first email cannot be unsaid. The saga **pivots**.

Six rules keep this from turning into folklore.

1. **Idempotency on do and on undo.** The key is `saga_id + step + direction`. Retries will happen. A timeout does not mean the commit failed to land. A refund that runs twice is this key, missing. The key lives with the step, not in a log line.
2. **Reservations, also called semantic locks,** when a visible half-state is too loud. Hold inventory as `HELD`, then convert it to `SOLD` or `RELEASED`. That small atomic story sits inside one owner and wraps an eventual saga outside.
3. **Forward recovery** when finishing is cheaper than undoing. Retry dispatch. Do not refund a charge you can still fulfill.
4. **Timeouts are transitions.** When the deadline fires, the machine moves. A log line is not a transition.
5. **Park and human** when the compensation cannot run: a partner API, or a side effect that cannot be reversed. A saga that cannot fail safe has to fail loud.
6. **Leave derived views alone.** If search indexed a ticket, rebuild the index from the system of record. Compensations are for other systems of record, not for copies.

The failure modes sit on those rules. A compensation written as if it cannot fail. A compensation that shares no id with the original step. A conductor that retries `do` after `undo` already ran. Treating a 2PC rollback as a compensation: different protocol, different visibility. An irreversible step — the email you cannot unsay, the partner call you cannot reverse — does not get a fake undo. The machine pivots, parks for a person, or the writes are merged back into one store.

"The log will handle the saga" is not a cell on the cube. A broker can carry `ReserveFailed`. It does not choose atomic versus eventual, it does not own the join, and it is not the state store. Events are a pipe.

## Recap

A saga is a state machine plus an undo story for a plot whose steps commit in more than one store. Place it on three axes: whether half-writes are visible, who drives the next step and the undo, and whether steps wait or fan out. The eight cells are those axes with names — Epic, Phone Tag, Fantasy Fiction, Horror Story, Fairy Tale, Time Travel, Parallel, Anthology — and the letter `a` means parallel in the first position and atomic in the second. Atomic cells hold or prepare and tend to become distributed commit; eventual cells commit locally and compensate with new business operations. Name the cell, write one compensation per local commit, key do and undo by saga, step, and direction, and put the state row on the design. A pipe does not do that work.

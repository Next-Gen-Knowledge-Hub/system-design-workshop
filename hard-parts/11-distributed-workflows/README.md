# 11. Managing distributed workflows

These notes teach how a business plot runs across services, from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## A click becomes a plot

Inside one process, opening a ticket is one function call. After that process is split, the same click is a plot that crosses services: write the ticket, reserve a technician, charge a card, send a notice. Support's question is no longer "which function threw?" It is "we are on step 3 of 5 — where is that sentence stored, and who is allowed to know the next step?"

That pair is the whole design: **style** and **state**. Style is who may know the next step. State is the record of which step this particular ticket has reached. The product that carries the bytes is a separate choice. A log, a queue, and an HTTP call are pipes. None of them picks the style. A queue of commands can still be orchestration. An HTTP call can still be orchestration. Publishing a fact can still be choreography. Judge the direction of knowledge.

An **architecture quantum** is the set of parts that ship and fail together. Workflow style is allowed to change at the boundary between quanta. It becomes unreadable when it changes on every arrow.

Two kinds of coupling show up in either style. **Static coupling** means one service's source names the next worker, so a change to the plot changes that source. **Semantic coupling** means services share the meaning of a fact and the order of the story, even when no source file names its neighbor. Counting topics does not measure either one. Who must change together does.

## Orchestration

After the split, nobody can see the plot. Support asks four teams. Timeouts fire in four places. A new step, such as a fraud check, means hunting every service that used to know the order.

Orchestration puts a **conductor** in front of the participants. The conductor accepts the client's request, or an initial command. It calls participants in an order it owns. It writes a **workflow row**: workflow id, current step, each participant's status, a deadline, and correlation ids. Timeout, retry, and "we are done" run in that one process. Participants stay **plot-dumb**. Billing knows how to charge. Dispatch knows how to assign. Neither stores "I am step 2 of ticket-open."

Walk one ticket.

1. The client asks to open a ticket. The conductor creates workflow `w-1842` at step `open`.
2. The conductor tells the ticket service to write the ticket. The ticket service writes its own row and replies. It does not know that a charge comes later.
3. The conductor tells dispatch to reserve a technician. Dispatch assigns and replies.
4. The conductor tells billing to charge the card. Billing charges and replies.
5. The conductor marks `w-1842` complete.

```
client
  |
  v
conductor -- open ticket --> ticket service
  |  ^
  |  +---- assign --------> dispatch
  |  ^
  |  +---- charge --------> billing
  v
workflow row: w-1842 | step | status | deadline
```

The row is updated after each reply, so a crash has a step to resume from. The calls may be synchronous HTTP, or commands on a queue. `ChargeCard` on a queue is still orchestration, because the conductor addressed billing by name and knew that billing was next. The protocol changed. The direction of knowledge did not. This shape is hub-and-spoke, which is also the sequence people draw once they have been paged and could not find the ticket.

The failure mode is the **god service**. The conductor learns every domain. It deploys whenever any step changes. It sits on the latency path of every customer action. Billing and dispatch look independent of each other, and both depend on the hub: when the hub is down, the plot is down, even while billing and dispatch are healthy. If the hub grows until it is the old monolith reached through network hops, the split was paid for twice.

## Choreography

Choreography has no service whose job is the plot. Participants **react to facts** and emit facts. The plot is the **graph of reactions**.

The same ticket:

1. The ticket service writes its row and publishes `TicketOpened`.
2. Dispatch listens, reserves a technician, and publishes `TechAssigned`.
3. Billing listens for the fact it cares about and charges the card.
4. The notifier listens and sends the message.

```
client -> ticket service --TicketOpened--> dispatch --TechAssigned--> billing
                                 \                         \
                                  +-----------------------> notifier
```

On the happy path, deploy independence is real. The notifier can add an SMS listener without a change that makes the ticket service call it. Static coupling to a hub drops, because the ticket service does not name its subscribers. Semantic coupling stays. Someone still decided "after open, assign." That decision now lives in every subscriber, in topic names, in YAML, and in what the team remembers.

The failure mode is an undrawable plot. A new engineer finds eight listeners and a wiki, and cannot point at a workflow, because there is no workflow object. Inserting a fraud check before the charge means coordinating every reaction that assumed the old order. A duplicate subscriber charges the card twice. A missed subscriber drops a step, and no on-call rotation owns "this customer journey."

A second failure is a command dressed as a fact. A message named `ChargeThisCard` that orders billing to act is orchestration with the order hidden. A fact named `CardCharged` says the charge already happened. If a downstream step must run for the business to be correct, that step is part of the plot. A **derived view** is a different job: a copy computed from a **system of record**, such as a search index filled by change-data capture (a feed of row changes) from ticket facts. The system of record is where the fact is written as truth. Derive the view from that record. Writing the truth and the copy by hand — a dual-write — makes two writers. A derived view is allowed to lag. Event time, the timestamp carried on the fact itself, matters when you treat the log as a data system and fold it into that view. A required step is not an optional reaction, and a search index filled from the log is not a workflow.

## Where the current step lives

Support, a timeout, and "did we already charge?" ask the same question: where is this instance? After a crash, a retry, or a click that finished halfway, some store has to know.

Name the store. It is not the ticket table. A workflow design can answer, in one sentence, where this ticket's step lives.

The **ticket row** is a domain fact: id, customer, the ticket's own status. The ticket service owns it. The **workflow row** is progress of the plot: workflow id (the id of this run), step, retries, deadline. In orchestration the conductor owns that row. The conductor is a state machine you can query, test, and expire. Mixing the two — a column such as `awaiting_payment` that five services update — is joint ownership by accident. Joint ownership means more than one service writes the same fact, so none of them can change it alone.

In choreography, progress is scattered. Each participant keeps an "I have seen this id" record. The global step is an emergent property. You either reconstruct it from the events, or you accept that support cannot be answered in one query. Teams that tire of that add a **tracker**: a service that listens to every fact and writes a progress row. The tracker sends no commands. It is still a conductor. The state owner returned through the side door, and everyone who asks "where is this ticket?" now depends on it.

A broker stores messages. It does not store "step 3 succeeded and step 4 is waiting on a human." You can compute that sentence from an event log when three things are true: the log is kept with retention long enough to be the system of record, order is preserved, and a projection job folds the log into a row. Until that job exists, support searches logs (the usual habit is to grep Kibana).

**Idempotency** sits beside the state. The conductor retries. A listener is delivered the same message twice. If "charge" is not keyed by `workflow_id + step`, the retry bills the customer twice. State without that key is a retry storm with a dashboard.

The timeout policy lives with the state owner. "We gave up" is a transition in the machine. A retry setting on the broker does not write that transition. If nobody owns the state, nobody owns the give-up.

## Which owner you can stand

"Orchestration versus events" is a bundled argument. The dimensions that actually move are visibility of the plot, which deploys are glued together, how failure is handled, latency, and who is paged at 3 a.m. Trade the state owner.

| What you need | Where the state tends to live |
|---|---|
| One query for "where is this?" | A conductor, or a tracker you treat as a hub |
| Timeout, compensate, and give-up in one place | A conductor |
| Listeners that ship without a hub release | The reaction graph |
| A short linear plot that changes often | A conductor, so one machine changes |
| A cloud of optional reactions | Choreography, because it is not one machine |
| A participant that must not know its neighbors | A conductor |
| A hub that would become the whole company | Choreography, plus a way to see the graph |

A **compensation** is a later business action that undoes a step already committed, such as a refund. Orchestration gives the timeout, the compensation, and the give-up one place to run.

Orchestration concentrates static coupling on the conductor. A participant is replaceable when the command and the reply stay the same. The conductor's quantum grows with every plot it owns, because those plots ship and fail with the hub. Split conductors by plot family — checkout, onboarding, refund — so one ConductorService is not the firm.

Choreography lowers static coupling to a hub and raises semantic coupling through event contracts. A fat event that carries the whole ticket is the plot stuffed into a payload. That payload is a **stamp**: every consumer depends on the whole document. Changing the plot changes the stamp everyone already consumed.

The state owner is the coupling hub even when the drawing hides it. A workflow engine — Camunda, Temporal, Step Functions, or a homegrown table — is an orchestrator with stronger persistence. If the engine addresses the next worker, it is a conductor with a nicer log, including when the messages it emits look like events. A notification mesh with an implicit order is choreography with weaker persistence. Pick the owner, then pick the product.

Three hybrids are ordinary.

Orchestrate the path that must be correct, and choreograph the side effects. Charge and assign are the plot: both have to happen. Indexing search, tweeting, and training a model are derived reactions; they may lag. A notice that must not roll back the charge is in that second group: the charge and the assignment stay on the conductor, and the notice is a reaction. Run those two jobs on different topics and different on-call rotations.

Apply **granularity** to the hub. Granularity is how much behavior sits in one service. One conductor per plot family is that split. A chatty orchestrated plot — many small calls that always travel together — is a vote to merge those services into one quantum. Adding another conductor does not fix a split that was too fine. That case is a granularity bug. A style bug is different: one conductor for a cloud of optional reactions, or a hub that has become the company, or a workflow engine described as choreography while it still names the next worker.

Change style at a quantum boundary. Domains can choreograph between each other while one domain orchestrates the several steps it still owns. Changing the rule on every other arrow means nobody can remember which arrow is which.

The **out-of-context trap** is importing a conclusion from a shape you do not have. A large company's event mesh, copied into a ten-person shop, is one case. A single BPMN process engine — one central executable model of every process — copied into a company whose plots are independent notifications, is the other. In both, the coupling metric people reach for is the number of topics. The metric that matches the design is who must change together.

## Recap

A distributed workflow is a plot across services and a named place where this run's step lives. Orchestration puts the next step in a conductor that addresses workers by name and stores the workflow row. Choreography puts the next step in reactions to facts, and the global step stays scattered until a tracker brings the state owner back. Required steps are the plot. Derived views may lag. The owner of the workflow state is the coupling hub, and that owner is who times out, retries, and gives up.

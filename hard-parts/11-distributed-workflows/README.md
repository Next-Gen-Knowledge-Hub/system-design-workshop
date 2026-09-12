# 11. Managing distributed workflows

Companion notes for **Chapter 11** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

Once you split a monolith, a customer action is no longer one function
call. It is a **plot** that crosses services: open a ticket, reserve a
tech, charge a card, send a notice. This chapter is who is allowed to
know that plot, and where the sentence "we are on step 3 of 5" lives.
Skip it and you will buy a message bus, call the result "decoupled,"
and then spend a year asking every team where a ticket went.

Sagas, compensations, and the three-letter catalog are
[ch. 12](../12-transactional-sagas/). This folder stops at **style and
state**. Ownership of rows is [ch. 9](../9-data-ownership/). How you
*read* another service's data is [ch. 10](../10-distributed-data-access/).

## See also (do not merge)

**Brokers, CDC, event time** — how a log is a data system — are
[DDIA ch. 12](../../ddia/12-stream-processing/). **Derive, don't
dual-write** (systems of record versus derived views) is
[DDIA ch. 13](../../ddia/13-streaming-philosophy/). Neither chapter
picks **orchestration versus choreography** as a business-process
style. Kafka does not make you choreographed. HTTP does not make
you orchestrated. The pipe and the plot are different jobs.

## The mental model

Two ways to run a plot. Same services. Different owner of the story.

```
  ORCHESTRATION                         CHOREOGRAPHY

  client                                    client
    |                                         |
    v                                         v
 [ CONDUCTOR ] --do this--> [ A ]         [ A ] --happened--> [ B ]
    |   ^                   [ B ]           |                  |
    |   | replies           [ C ]           +--happened--> [ C ]
    |   +---------------------              (no boss; the plot
    v                                        lives in reactions)
  workflow row:
  id | step | status | due
```

The one sentence to remember a year from now: **a workflow has a
state owner, and that owner is a coupling hub** — whether you drew a
conductor box or pretended the events would remember for you.

Two consequences fall out of the diagram. First, "we are event-driven"
is not a style. Events are a transport. Style is **who is allowed to
know the next step**. Second, if you cannot answer "where does this
ticket's step live?" in one sentence, you do not have a workflow
design. You have hope.

## Orchestration communication style

**Problem** — After the split, nobody can see the plot. Support asks
four teams. Timeouts fire in four places. A new step (fraud check)
means hunting every service that used to "just know."

**Solution** — Put a **conductor** in front of the participants. The
conductor:

- accepts the client's request (or an initial command),
- calls participants **in an order it owns**,
- records **workflow state** (which step, which status, when to give
  up),
- handles timeout, retry, and "stop, we are done" in one process.

Participants stay **plot-dumb**. Billing charges. Dispatch assigns.
Neither service knows it is "step 2 of ticket-open." That knowledge
is the conductor's job. Calls may be synchronous HTTP or asynchronous
commands on a queue. The style is the same: **the conductor addresses
the next worker by name.**

This is hub-and-spoke. It is also how most humans already draw the
sequence diagram on a whiteboard, which is why it is the default once
someone has been paged.

**Failure** — The conductor becomes a **god service**: it knows every
domain, deploys when any step changes, and sits on the latency path
of every customer action. Participants look "decoupled" from each
other and are tightly coupled **to the hub**. If the hub is down, the
plot is down — even if Billing and Dispatch are healthy. If you grow
the hub until it *is* the old monolith with extra network hops, you
paid twice.

Orchestration is not "RPC is bad, messages are good." An orchestrator
that emits *commands* (`ChargeCard`) is still orchestrating. An
orchestrator that waits on HTTP is still orchestrating. Judge the
**direction of knowledge**, not the protocol.

## Choreography

**Problem** — The hub is the bottleneck you split the monolith to
escape. Every new plot step is a conductor release. Teams cannot
ship a participant without a hub change. The conductor's source tree
is a map of the whole company.

**Solution** — Participants **react to facts** and emit facts. Open
ticket writes its row and publishes `TicketOpened`. Dispatch listens,
assigns a tech, publishes `TechAssigned`. Notify listens to that.
There is no service whose job is "know the plot." The plot is the
**graph of reactions**.

What you gain is deploy independence on the happy path: Notify can
add an SMS listener without asking the ticket service to call it.
What you paid for in [ch. 2](../2-coupling/) as *static* hub coupling
drops. *Semantic* coupling does not. Someone still designed "after
open, assign." That someone is now **every subscriber**, and the
design lives in YAML, topic names, and tribal memory.

**Failure** — The plot becomes **undrawable**. A new engineer cannot
find the workflow because it is not a workflow. It is eight listeners
and a wiki. Changing the order (fraud check *before* charge) means
coordinating every reaction that assumed the old order. Duplicate
subscribers double-charge. A missed subscriber silently drops a
step. "Who owns this customer journey?" has no on-call rotation.

Choreography also tempts you to put **commands in fact clothing**.
`ChargeThisCard` published as if it were `CardCharged` is orchestration
with extra denial. If a downstream *must* run for the business to
be correct, you have a plot, not a derived view. Derived views can
lag; plots cannot pretend they are optional. That distinction is
exactly why this folder does not merge with
[DDIA ch. 13](../../ddia/13-streaming-philosophy/).

## Workflow state management

**Problem** — Support, timeouts, and "did we already charge?" all
need the same question: **where is this instance?** After a crash,
a retry, or a half-finished click, something must know.

**Solution** — Name the state store on purpose.

In orchestration, state is usually a **row the conductor owns**:
`workflow_id`, current step, statuses of participants, deadline,
correlation ids. The conductor is a state machine. You can query
it. You can test it. You can expire it. That is the whole appeal.

In choreography, state is **scattered**. Each participant has its
own "I have seen this id" record. The global step is an emergent
property: reconstruct it from events, or accept that you cannot
answer Support in one query. Teams that get tired of this add a
**tracker** (a service that listens to everything and writes a
progress row). Notice what you just did: you built a conductor
that does not send commands. The state owner returned through the
side door.

State is not "the database of record for tickets." Ticket rows are
[ch. 9](../9-data-ownership/). Workflow state is **progress of the
plot**: steps, waits, retries. Mixing them — storing "awaiting
payment" as a column that five services update — is how you get
joint ownership by accident.

```
  ticket row (domain fact)          workflow row (plot progress)
  -----------------------           ---------------------------
  id, customer, status              saga/workflow id
  owner: Ticket service             step, retries, deadline
                                    owner: conductor OR "nowhere"
```

**Failure** — Two common lies. One: "the message broker is the
state." Brokers remember **messages**, not "step 3 succeeded and
step 4 is waiting on a human." Two: "we will compute state from
the event log when we need it." You can, if you treated the log as
a system of record with retention, order, and a projection job —
that is [DDIA ch. 12](../../ddia/12-stream-processing/) work, and
you still have to *build the projection*. Until you do, Support
greps Kibana.

Idempotency sits next to state. The conductor retries; the listener
redelivers. If "charge" is not keyed by `workflow_id + step`, you
double-bill. State without idempotency is a retry storm with a
dashboard.

## Trade-offs: state owner and coupling

**Problem** — Reviews argue "orchestration vs events" as a moral
choice. The dimensions are entangled: visibility, deploy coupling,
failure handling, latency, and **who gets the 3 a.m. call**.

**Solution** — Trade the **state owner**, not the fashion.

| You want | Lean toward |
|---|---|
| One query for "where is this?" | Orchestration (or a tracker you admit is a hub) |
| Timeout, compensate, give-up in one place | Orchestration — [ch. 12](../12-transactional-sagas/) is easier here |
| Teams ship listeners without a hub release | Choreography |
| The plot is short, linear, and changes often | Orchestration (change one machine) |
| The plot is a cloud of optional reactions | Choreography (do not pretend it is a single machine) |
| A participant must not know its neighbors | Orchestration |
| A hub would become the company | Choreography, plus a way to *see* the graph |

Coupling, from [ch. 2](../2-coupling/):

- **Orchestration** concentrates **static** coupling on the
  conductor. Participants are easy to replace *if* they keep the
  same command/reply contract. The conductor's architecture
  quantum grows with every plot it owns. Split conductors by
  *plot* (checkout vs onboarding), not one ConductorService for
  the firm.
- **Choreography** lowers static coupling to a hub and raises
  **semantic** coupling through event contracts. You will feel
  that in [ch. 13](../13-contracts/): a "fat" event that carries
  the whole ticket is a plot in a payload. Changing the plot
  changes the stamp everyone already consumed.

The state owner is the coupling hub **even when you hide it**. A
"workflow engine" (Camunda, Temporal, Step Functions, homegrown
table) is an orchestrator with better persistence. A "notification
mesh" with an implicit order is choreography with worse
persistence. Pick the owner. Then pick the product.

Hybrid is allowed and normal: **orchestrate the money path,
choreograph the side effects**. Charge and assign are a plot.
"Also index search, also tweet, also train a model" are derived
reactions. Do not run those two jobs on one topic and one on-call.

A second hybrid shows up as **scale**: one conductor per *plot
family* (checkout, onboarding, refund), not one ConductorService
for the firm. That is granularity ([ch. 7](../7-service-granularity/))
applied to the hub itself. A third: choreography among domains,
orchestration **inside** a domain that still owns several steps.
The style can change at a quantum boundary. It should not change
every other arrow, or nobody can remember the rule.

**Failure** — The out-of-context trap ([ch. 15](../15-trade-off-analysis/)):
copying a big-tech event mesh into a ten-person shop, or copying
a single BPMN engine into a company whose plots are actually
independent notifications. Also: measuring "decoupling" by number
of topics. Topics are not a coupling metric. **Who must change
together** is. Also: calling a workflow engine "choreography"
because it emits events. If the engine addresses the next worker,
it is a conductor with a nicer log.

## How this shows up when you design something

- Draw the plot as a sequence. Circle the box that is allowed to
  know the next step. If you circled eight boxes, you chose
  choreography. Say so.
- Write down the state store: table, engine, or "reconstruct from
  log" (and who builds the reconstruction).
- For each step, mark **required for correctness** vs **optional
  reaction**. Required steps are a workflow. Optional ones may be
  derived data ([DDIA 13](../../ddia/13-streaming-philosophy/)).
- Do not cite the broker as the architecture. Cite the style.
- Granularity warning from [ch. 7](../7-service-granularity/): a
  chatty orchestrated plot is a vote to **merge** services, not a
  vote to add a conductor.
- Timeout policy lives with the state owner. If nobody owns state,
  nobody owns "we gave up." That is not a retry setting on the
  broker; it is a transition you forgot to write.

## Check yourself

1. A team says "we switched to events, so we are choreographed."
   Name one design that uses events and is still orchestration.
2. Who is the state owner in your last cross-service user action?
   If the answer is "Kafka," what question can you still not
   answer for Support?
3. Why might adding a "workflow tracker" that only listens (never
   commands) re-create orchestrator coupling?
4. Ticket-open must charge, then assign, then notify. Notify
   failing must not roll back the charge. Which steps are plot,
   which are reactions, and which style fits each?
5. Explain static vs semantic coupling for the two styles without
   using the words "tight" or "loose."
6. An orchestrator calls services over a queue, not HTTP. Did the
   style change? What did?
7. DDIA talks CDC into a search index. Is that a distributed
   *workflow* in this chapter's sense? Why or why not?
8. You need to insert a fraud check between assign and charge.
   Walk the change in both styles. Who deploys?
9. When is a god-orchestrator a granularity bug ([ch. 7](../7-service-granularity/))
   rather than a style bug?
10. "The log is the workflow." What extra machinery would make
    that sentence true, and which book owns that machinery?

Continue to [Transactional sagas](../12-transactional-sagas/).

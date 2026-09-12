# 13. Contracts

Companion notes for **Chapter 13** of *Software Architecture: The Hard
Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

You split the monolith. You named owners ([ch. 9](../9-data-ownership/)).
You picked a workflow style ([ch. 11](../11-distributed-workflows/)).
None of that survives contact with **the payload**. The contract is
the seam: what a producer is allowed to say, and what a consumer is
allowed to assume. Skip this chapter and you will "decouple" services
that all break when Billing adds a field to a JSON blob they never
read.

## See also (do not merge)

**How bytes evolve** — JSON vs Protobuf vs Avro, backward and forward
compatibility, rolling upgrades, schema registries — is
[DDIA ch. 5](../../ddia/5-encoding-and-evolution/). This folder is
**how much domain you leak** across a service boundary: strict vs
loose agreement, and **stamp coupling** (fat documents that glue
teams together). You need both. A perfect Avro pipeline that
publishes the entire Customer aggregate is a compatibility success
and an architecture failure.

## The mental model

```
  producer                         consumer
     |                                ^
     |         CONTRACT (the seam)    |
     +---- typed fields? or a bag? ---+
     |                                |
     |   "how much of our world       |
     |    did we just ship?"          |

  stamp coupling (the trap):

  [ Customer { 40 fields } ] ----event----> [ Billing ]
                                             needs customer_id
                                             + card_ref
                                             now breaks when
                                             Address.line2 changes
```

The one sentence to remember: **the contract is coupling you chose
on purpose**; a fat payload is coupling you chose by accident.

Two consequences. First, "we use Avro" does not decide strict vs
loose in this chapter's sense — encoding can carry a tiny contract
or a dump of the database. Second, consumers that "just ignore extra
fields" are still coupled if a rename, a meaning change, or a nested
object they *do* touch moves. Ignoring is not insulation.

## Strict versus loose

**Problem** — Teams need to change independently. They also need to
**understand each other**. Those wants fight in the payload.

**Solution** — Place the contract on a spectrum, then pick a point
per seam (not per company).

**Strict** means the agreement is **explicit and checkable**: names,
types, required vs optional, enumerations, versions. Breaking
changes fail at generate, compile, contract-test, or broker schema
check — *before* a customer request. gRPC + Protobuf, Avro with a
registry and a compatibility mode, OpenAPI with required fields and
consumer-driven tests: different encodings, same instinct. The
producer is not allowed to "just add a required field." The consumer
is not allowed to magically know a field you never promised.

**Loose** means the agreement is **a bag plus conventions**: JSON
objects with whatever keys seemed useful, "extra fields are fine,"
stringly types, implicit enums. Easy to emit. Easy to add a key at
2 a.m. The consumer picks the two fields it wanted and hopes they
still mean the same thing on Monday.

Strict is not "good" and loose is not "startup-speed." Strict costs
**coordination**: a required-field change is a project. Loose costs
**discovery in production**: a missing key, a null that used to be
a string, a reused field with a new meaning.

A useful middle: **strict about the few fields that mean money or
identity, loose about decoration** — but then decoration must not
be how the next workflow step makes a decision. The moment Notify
parses an optional nested `customer.tier` to choose a template, that
nest is part of the contract, however optional the README says it is.

**Failure** — One company-wide rule. Public APIs, pub/sub plots, and
an internal DTO between two modules of the same quantum do not want
the same tightness. Also: calling GraphQL "loose" because clients
pick fields. The *schema* can still be strict; the *stamp* can still
be huge if the client asks for the world.

## Trade-offs

**Problem** — Reviews polarize: "without a schema we will die" vs
"schemas make us a waterfall." Both can be right for a different
seam.

**Solution** — Trade tightness against **change rate, blast radius,
and how you find out you were wrong.**

| Tighter (strict) | Looser |
|---|---|
| Breaks at build / codegen / registry | Breaks in prod, often in *one* consumer |
| Versioning is a conversation | Adding a key feels free |
| New consumers onboard from a spec | New consumers onboard from an example payload that lies |
| Slow, visible coordination | Fast producer, hidden consumer coupling |
| Good when a wrong field spends money | Good when the payload is a log line nobody parses |

Versioning is not a third axis so much as the **strict** world's
release valve: additive optional fields, new event types instead of
reusing names, explicit `v2` when meaning changes. The loose world
*also* versions — accidentally — every time a producer ships a new
shape. You just do not have a word for it until an incident.

Consumer-driven contracts (tests the consumer publishes, the
producer runs) are how strict stays honest without a central
committee for every field. They do not replace choosing *how much
to expose*. A consumer-driven test that still asserts the whole
Customer aggregate will *preserve* stamp coupling with extra CI
green.

**Failure** — Strict schemas that copy the **database table**. You
did not tighten the contract. You **exported the schema** and now
every column rename is a fleet event. Loose bags that become the
workflow token (the whole ticket JSON walks through six services).
You did not stay flexible. You built Phone Tag
([ch. 12](../12-transactional-sagas/)) inside a hashmap.

## Contracts in microservices

**Problem** — Inside a monolith, a Java type was the contract and
the compiler was the broker. Across processes, nothing fails until
runtime unless you install a substitute for the compiler.

**Solution** — Treat every crossing of an architecture quantum
([ch. 2](../2-coupling/)) as a **published** agreement:

- **Commands** (orchestrator → participant): usually *stricter*.
  They are imperative; missing fields are bugs; they should contain
  **intent + ids**, not a mirrored customer record.
- **Replies / queries**: return what the caller named in the
  contract, not your internal aggregate.
- **Events** (facts): name a **happened** thing. Thin: ids + the
  fields that *define* the fact. If listeners need more, they
  **ask the owner** ([ch. 10](../10-distributed-data-access/)) or
  subscribe to a projection built for them — that is a new contract,
  not a reason to stamp the OLTP row.
- **Synchronous public APIs**: version in the open; treat
  compatibility as product.
- **Same-quantum internals**: do not pretend they are
  microservices contracts. If two modules always deploy together,
  a shared type is allowed. Calling it a "REST contract" is costume.

Ownership ([ch. 9](../9-data-ownership/)) shows up here: **only the
owner should emit the canonical fact** about a field. If Dispatch
echoes `customer.address` it heard from Ticket, Dispatch has started
a second, stale contract for address. Stamp coupling *creates*
shadow ownership.

Wire format still matters — that is DDIA 5. Pick Avro/Protobuf when
you want evolution rules the computer can check. Pick JSON when
humans and browsers are the consumers and you are willing to
**discipline** the shape yourself. The Hard Parts question is not
JSON vs Avro. It is **does this payload leak another team's
innards?**

**Failure** — "The OpenAPI spec is the contract" while producers
hand-edit JSON the spec does not mention. Specs that are not in CI
are wishes. Also: one "canonical event model" for the company that
every domain must fit. That is a warehouse of contracts, with the
same bottleneck as a central warehouse of tables
([ch. 14](../14-analytical-data/)).

## Stamp coupling

**Problem** — A consumer needs two fields. The producer already has
a struct with forty. Passing the struct is one line. It feels like
reuse ([ch. 8](../8-reuse-patterns/)). It is the most expensive
reuse in the book.

**Solution** — Name **stamp coupling**: coupling through a
**composite** the consumer did not want. The stamp can be a Java
object, a protobuf message with 90 fields, a JSON document, a
database row dumped to a topic, a "fat event," or an HTTP response
that returns the aggregate "for convenience."

Three bills, always together:

1. **Over-coupling.** Billing now changes when `Address.line2` is
   added, renamed, nested, or typed from string to object — even if
   Billing never prints an address. The consumer's **logical**
   need was `customer_id` + `card_ref`. The **actual** dependency
   is the producer's module boundary. Independent deploy was the
   point of the split; the stamp glues the deploys back together.
   Schema registries make this worse if they force every consumer
   to accept the new shape before the producer can ship: you
   automated the glue.
2. **Bandwidth.** Forty fields, nested orders, base64 blobs,
   "send the screenshot while we are here." You pay it on every
   message, every retry, every new subscriber who only wanted an
   id. At low volume this looks free. At topic-fanout it is a
   capacity plan you never wrote. Caching the stamp "so we don't
   call Ticket" multiplies copies
   ([ch. 10](../10-distributed-data-access/)).
3. **Workflow.** The stamp becomes the **token that is the plot**.
   Each step mutates the blob and forwards it. There is no
   conductor; there is a document walking the building. That is
   Phone Tag with extra fields. Changing the plot means changing
   the blob, which means changing every pair of hands that
   forwarded it. The document is also a **cache of other owners'
   data**, so compensations ([ch. 12](../12-transactional-sagas/))
   undo a stamp that may already be stale relative to the source.

What to do instead, without pretending one tactic fits every seam:

- **Carve** a contract per consumer job: `ChargeCustomer`
  contains ids and an amount, not `Customer`. Duplication of two
  fields is cheaper than coupling to thirty-eight.
- **Thin events + pull:** publish `TicketOpened { ticket_id }`;
  consumers that need more **query the owner** (runtime coupling,
  fresh data) or consume a **purpose-built projection**.
- **Fat events on purpose:** when the consumer *must* see a
  snapshot as it was at the moment of the fact (audit, "what did
  we bill against"), include that snapshot **and freeze its
  meaning**. That is a versioned type, not "here is our ORM
  entity."
- **Do not** "fix" stamp coupling by sharing the same DTO jar
  across twelve services. You moved the stamp into a shared
  library ([ch. 8](../8-reuse-patterns/)) and bought version hell.
- **Version the meaning, not the dump.** `TicketOpened.v2` that
  adds `priority` is a conversation. `TicketOpened` that grows a
  nested `customer` "because Notify asked once" is a stamp with
  a changelog.

**Failure** — Measuring cleanliness by "one event type per
aggregate." That metric *maximizes* stamps. Measuring cleanliness
by "events are ids only" and then watching every consumer hammer
the owner on the hot path: you traded stamp coupling for
**chatty workflow**, which [ch. 7](../7-service-granularity/)
already warned is a reason to merge or to build a projection.
The trade-off is the point. Stamp vs chatter is a
[ch. 15](../15-trade-off-analysis/) analysis, not a slogan.

## How this shows up when you design something

- For each arrow on the diagram, write the **smallest** payload
  that still lets the other side do its job. If you cannot name
  the job, you are about to stamp "just in case."
- Mark fields that are **owned** here vs **echoed** from another
  quantum. Echoed fields are stamps until proven otherwise.
- Say how a breaking change is detected: compiler, registry,
  contract test, or a customer. If the last one, you chose loose
  *and* you chose your detector.
- Do not cite "schema-first" as if it answered leakiness.

## Check yourself

1. DDIA 5 vs this chapter: a team adds a field with a new Avro
   number. Compatibility holds. A downstream still pages. What
   kind of coupling did Avro not save them from?
2. Strict vs loose: where does each design *find out* it was
   wrong? Give one seam where you would pick the slower finder.
3. Commands vs events: why should a command usually be stricter
   than a "something happened" fact — or when would you invert
   that?
4. Billing needs two fields from Customer. You pass the Customer
   object. Name the three bills of stamp coupling on that one
   decision.
5. How does a fat event become a workflow token? Which saga cell
   from [ch. 12](../12-transactional-sagas/) did you accidentally
   pick?
6. Thin event + pull vs fat snapshot: pick one production case
   for each. What coupling did you accept?
7. Shared DTO jar across services: which reuse pattern from
   [ch. 8](../8-reuse-patterns/) is this, and what did you just
   do to deploy independence?
8. "Consumers ignore unknown fields, so we are decoupled." Give
   two changes that still break those consumers.
9. Who is allowed to emit `customer.address` as a fact? What
   happens if Dispatch echoes it on `TechAssigned`?
10. You tightened the contract by publishing the SQL schema of
    the owner as the event type. What did you actually couple?

Continue to [Managing analytical data](../14-analytical-data/).

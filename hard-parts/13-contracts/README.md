# 13. Contracts

These notes teach the seam between services — what a producer may say, and what a consumer may assume — from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## The payload is the coupling

Splitting a system names owners and picks a workflow style. The design still dies in the payload. Billing adds a field to a JSON document. Every consumer that never reads the field breaks anyway, because they all deserialize the same blob.

The **contract** is that seam: the fields, types, and meanings a producer is allowed to emit, and the assumptions a consumer is allowed to make. It is coupling chosen on purpose. A fat payload is coupling chosen by accident, because the struct was already there.

Two different questions get the same words. One question is how bytes evolve: JSON, Protobuf, or Avro; backward compatibility (new code reads old data) and forward compatibility (old code reads new data); rolling upgrades where old and new processes run together; schema registries that check a new schema before a producer may publish. The other question is how much of a domain crosses the boundary. A perfect Avro pipeline that publishes the entire Customer aggregate can pass the compatibility check and still be an architecture failure. "We use Avro" does not pick strict or loose. An encoding can carry a tiny contract or a dump of the database.

An **architecture quantum** is the set of parts that ship and fail together. A contract is what you publish when knowledge crosses from one quantum to another. Inside one quantum, two modules that always deploy together may share a type. Calling that shared type a REST contract is costume.

## Strict or loose, per seam

Teams need to change independently, and they need to understand each other. Those wants meet in the payload. Pick a point on the spectrum **per seam**. One company-wide rule is the failure mode: a public API, a plot carried by publish/subscribe, and an internal data-transfer object between two modules of the same quantum do not want the same tightness.

**Strict** means the agreement is explicit and checkable: names, types, required versus optional, enumerations, versions. A breaking change fails at code generation, at compile time, in a contract test, or at the broker's schema check, before a customer request. gRPC with Protobuf, Avro with a registry and a compatibility mode, and OpenAPI with required fields plus consumer-driven tests are different encodings with the same instinct. The producer may not "just add a required field." The consumer may not assume a field nobody promised.

**Loose** means a bag plus conventions. JSON objects gain whatever keys seemed useful. Extra fields are declared fine. Types are strings when they should have been enumerations (**stringly** values). Enums are implicit. Emitting is easy, and adding a key at 2 a.m. is easy. The consumer takes the two fields it wanted and hopes they mean the same thing on Monday.

Strict costs coordination: a new required field is a project, versioning is a conversation, and a new consumer onboards from a spec. Loose costs discovery in production: a missing key, a null that used to be a string, a field reused with a new meaning. Adding a key feels free, and a new consumer onboards from an example payload that lies. The producer ships fast, and the consumer coupling stays hidden until one consumer breaks. Strict is the slower finder, and it is the right finder when a wrong field spends money. Loose is a fit when the payload is a log line nobody parses.

A useful middle is strict about the few fields that mean money or identity, and loose about decoration. Decoration stops being decoration the moment a later step decides on it. If the notifier parses an optional nested `customer.tier` to choose a template, that nest is part of the contract, whatever a README says about optional.

Versioning is the release valve on the strict side: additive optional fields, a new event type instead of reusing a name, an explicit `v2` when the meaning changes. The loose side versions too, by accident, every time a producer ships a new shape. There is no name for that version until an incident.

**Consumer-driven contracts** are tests the consumer publishes and the producer runs. They keep a strict agreement honest without a central committee for every field. They do not choose how much to expose. A consumer-driven test that asserts the whole Customer aggregate preserves stamp coupling, and the continuous-integration build stays green while it does.

GraphQL does not land on the loose end just because a client picks fields. The schema can be strict. The stamp can still be huge if the client asks for the world.

Ignoring unknown fields is not insulation. A rename, a change of meaning, and a nested object the consumer does touch still break that consumer.

The other failure mode on this spectrum is a strict schema that copies the database table. That did not tighten a contract. It exported the schema, and every column rename is now a fleet event. The matching loose failure is a bag that becomes the workflow token: the whole ticket JSON walks through six services. That is **Phone Tag** inside a hashmap. Phone Tag is a plot with no conductor, where a token is passed from service to service and each hop mutates it. Changing the plot means changing the bag, which means changing every pair of hands.

## What each kind of message should contain

Inside a monolith, a Java type was the contract and the compiler was the broker. Across processes, nothing fails until runtime unless a substitute for the compiler is installed. Treat every crossing of a quantum as a published agreement.

**Commands**, from an orchestrator to a participant, are usually stricter. They are imperative. A missing field is a bug. They should carry intent and ids, not a mirrored customer record. `ChargeCustomer` contains ids and an amount.

**Replies and queries** return what the caller named in the contract, not the owner's internal aggregate.

**Events** name something that happened. A thin event carries ids plus the fields that define the fact. `TicketOpened` with `ticket_id` is that shape. Listeners that need more ask the owner, which is runtime coupling and fresh data, or they consume a projection built for them. The projection is a new contract. It is not a reason to stamp the operational row — the live **OLTP** row the customer-facing service writes — onto the event.

**Synchronous public APIs** are versioned in the open. Compatibility is part of the product.

**Same-quantum internals** stay internal. A shared type between modules that always deploy together is allowed.

Ownership shows up as who may speak. Only the owner emits the canonical fact about a field. If dispatch echoes `customer.address` that it heard from the ticket service, on an event such as `TechAssigned`, dispatch has started a second contract for address. That contract goes stale. Stamp coupling creates this shadow ownership.

Wire format still matters, and it is the byte-evolution question above. Avro or Protobuf is how you get evolution rules a computer can check. Adding a field under a new Avro field number can keep compatibility: old consumers still parse the message. JSON is how you serve humans and browsers, if you are willing to discipline the shape yourself. The architecture question is whether the payload leaks another team's innards. A compatible Avro change does not answer it. Downstream can still page, because the coupling was the domain in the document, not the field number.

Two specification failures sit here. An OpenAPI document that producers ignore, while they hand-edit JSON the spec does not mention, is a wish. A spec that is not in the build is a wish. And one canonical event model that every domain must fit is a warehouse of contracts: the same bottleneck as a central warehouse of tables, where every new fact waits on one team.

## Stamp coupling

A consumer needs two fields. The producer already has a struct with forty. Passing the struct is one line, and it feels like reuse. **Stamp coupling** is coupling through a composite the consumer did not want. The stamp can be a Java object, a protobuf message with ninety fields, a JSON document, a database row dumped onto a topic, a fat event, or an HTTP response that returns the aggregate "for convenience."

```
ticket service                         billing
  Customer { 40 fields } --event-->    needs customer_id + card_ref
                                       breaks when Address.line2 changes
```

Billing's logical need was `customer_id` and `card_ref`. The actual dependency is the producer's module boundary. `Address.line2` can be added, renamed, nested, or retyped from string to object, and billing changes even though billing never prints an address.

The stamp always sends three bills together.

1. **Over-coupling.** Independent deploy was the point of the split. The stamp glues the deploys back together. A schema registry makes this worse when it forces every consumer to accept the new shape before the producer can ship. The glue is now automated.
2. **Bandwidth.** Forty fields, nested orders, base64 blobs, "send the screenshot while we are here." You pay that on every message, every retry, and every new subscriber who only wanted an id. At low volume it looks free. At fan-out it is a capacity plan nobody wrote. Caching the stamp so a consumer does not call the ticket service multiplies copies of someone else's data.
3. **The plot itself.** The stamp becomes the token that *is* the workflow. Each step mutates the blob and forwards it. There is no conductor. A document walks the building. That is Phone Tag with extra fields. Changing the plot means changing the blob, which means changing every hop that forwarded it. The document is also a cache of other owners' data, so a **compensation** — a later business operation that tries to undo a committed step, such as a refund — undoes a stamp that may already be stale relative to the source.

What to do on a given seam:

- **Carve** a contract per consumer job. `ChargeCustomer` contains ids and an amount, not `Customer`. Duplicating two fields is cheaper than coupling to the other thirty-eight.
- **Thin events, then pull.** Publish `TicketOpened { ticket_id }`. A consumer that needs more queries the owner (runtime coupling, fresh data) or reads a purpose-built projection.
- **Fat events on purpose,** when the consumer must see the snapshot as it was at the moment of the fact: an audit, or "what did we bill against." Include that snapshot and freeze its meaning. That is a versioned type. It is not the ORM entity that maps the table.
- **Leave the shared DTO jar alone.** Shipping the same data-transfer object in a library used by twelve services moves the stamp into a shared library and buys version hell. Deploy independence goes with it.
- **Version the meaning.** `TicketOpened.v2` that adds `priority` is a conversation. `TicketOpened` that grows a nested `customer` because the notifier asked once is a stamp with a changelog.

The failure mode is a cleanliness metric that maximizes the stamp, or one that creates a different pain. "One event type per aggregate" pushes the whole aggregate onto the wire. "Events are ids only," followed by every consumer hammering the owner on the hot path, trades the stamp for a **chatty workflow**. Chatty, here, means a plot made of many small calls that always happen together. That is a reason to merge the services, or to build a projection. **Granularity** is how much behavior sits in one service; a chatty plot is a granularity problem. Stamp versus chatter is a trade-off with named costs on both sides. It is not a slogan, and "schema-first" does not answer whether the schema leaks.

For each arrow, write the smallest payload that still lets the other side do its job. If the job has no name, the payload is about to be stamped "just in case." Mark fields this quantum owns, and fields echoed from another quantum. Echoed fields are stamps until something proves otherwise. Say how a breaking change is detected: the compiler, a registry, a contract test, or a customer. Detection by the customer means the seam is loose, and the detector was chosen along with it.

## Recap

A contract is the coupling you publish at a quantum boundary: names, types, and how much of the domain is allowed to cross. Strict seams fail at build, generation, or a registry; loose seams fail in production, often in one consumer. Commands tend to be strict and thin (intent and ids). Events name a fact that happened; listeners that need more pull from the owner or from a projection. Stamp coupling is handing over a composite the consumer did not want, and it bills you in deploys, bandwidth, and a document that becomes the plot. A compatible encoding can still stamp. Carve the payload to the job, and only the owner emits the canonical fact.

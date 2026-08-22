# 5. Encoding and evolution

Companion notes for **Chapter 5** of *Designing Data-Intensive Applications*
(2nd ed.). Data is not “a JSON in memory.” It is bytes on the network
or disk, decoded by **some other version** of your code. This chapter
is how systems change without stopping: rolling upgrades, compatible
schemas, and the pipes data flows through.

## What this chapter is for

Maintainability (ch. 2) at the byte level. You want **rolling upgrades**:
new code on a few nodes, old code still running, no flag day. That only
works if:

- **Backward compatibility** — new code can read old data.
- **Forward compatibility** — old code can read (or at least tolerate)
 new data, usually by ignoring unknown fields. if the record is decoded
 into a model object that does not explicitly preserve unknown fields, the
 new field can be silently lost (data loss risk).

Every encoding format either helps you here or sets a trap.

## Formats for encoding data

**Language-specific** (Java serialization, Python pickle, Ruby Marshal):
fast to type, bound to one language, historically full of security
holes (decode as remote code execution), and weak at compatibility.
Do not put these on a public wire or a long-lived disk file.

**JSON, XML, CSV:** everywhere, human-readable, optional schemas. JSON
is the default for HTTP APIs. Costs: verbose, ambiguous numbers
(integers vs floats vs 64-bit ids that JavaScript cannot hold), no
native binary, CSV’s quoting hell. Compatibility is *how you use it*:
never reuse a field for a new meaning; add fields; be liberal about
unknown keys. this encoding schema's are not backward neigher forward
compatible, doesn't designed for that purpose. drops unknown fields and
coudn't reuse fields for different meanings.

**Binary JSON cousins** (MessagePack, BSON, CBOR): smaller, still
schemaless.

**Protocol Buffers, Thrift, Avro:** schema-driven binary.

- **Protobuf / Thrift:** field numbers. Add optional fields with new
 numbers; old readers skip unknown numbers. Do not change types in
 place. Generated code in many languages. gRPC rides on Protobuf.
 fully backward and forward compatible, old versions discard and don't modify
 new data, and new versions could read and modify old data.
- **Avro:** schema on write, often with a **schema registry**. Especially
 at home in Hadoop/Kafka land. Writer schema + reader schema are
 resolved against each other; that resolution *is* the compatibility
 story.

Schemas are documentation the compiler can check. They also let you
generate code and evolve safely. The downside: you cannot `cat` a
Protobuf on the terminal; you need tooling. That is a fair trade for
anything that will live longer than a hack week.

**Merits of schemas:** they make compatibility an explicit review
(“this change is backward compatible”) instead of an accident in prod
at 3 a.m.

## Modes of dataflow

Encoding only matters when data **moves**.

### Through databases

Process A writes a row; process B reads it later — maybe years later,
on new code. The database is an integration point. Unknown fields must
survive a read-modify-write: if B loads JSON, drops keys it does not
know, and writes back, it **strips** fields the new producer added.
That bug is classic. Same for ORM models that only map known columns.

### Through services: REST and RPC

Client encodes a request; server decodes, encodes a response; client
decodes. REST+JSON is simple and cache-friendly. RPC (gRPC, SOAP
legacy, Finagle, …) feels like a function call and hides the network
until a timeout reminds you it is not a function call (ch. 9).

Compatibility assumption that is nicer than databases: you can often
**upgrade servers first, then clients**, so you need backward
compatibility on requests and forward compatibility on responses. Mobile
apps break that assumption — old clients linger for years.

### Durable execution and workflows (2nd edition)

Long-running business processes (onboard a merchant, settle a payout,
retry a KYC check for three days) are not a single RPC. **Workflow
engines** (Temporal, Cadence, and cousins) persist each step so a crash
does not lose “we already charged the card.” That stored state is
*encoded data* that must survive **code deploys** while a workflow is
open. Treat workflow inputs/results with the same compatibility rules
as an event schema. If you invent a homegrown state machine in Redis,
you have the same problem without the tooling.

### Event-driven architectures

Producers write messages; consumers read them later, independently.
Message brokers and logs (ch. 12)
are the usual pipes. Compatibility is harder than RPC: you cannot
assume consumers upgraded first. **Schema registry + Avro/Protobuf**
exists for this. If a consumer republishes to another topic, it must
**forward unknown fields** or it becomes the strip-on-write bug again.

**Actor systems** (Akka, Orleans, Erlang) are message passing with
location transparency. Messages still need compatible encodings across
rolling upgrades. The network is allowed to drop messages; that is
already in the model, which is why actors map to distribution better
than “RPC that pretends to be local.”

## How this shows up when you design something

- Public HTTP API: JSON, version in the URL or in fields you only add,
 never mutate.
- Internal high-QPS services: gRPC + Protobuf, server-first deploys.
- Kafka pipelines: Avro/Protobuf + registry, compatibility checks in CI.
- Mobile: forever-forward-compatible responses.
- Workflows: version the workflow definition; never rename a field
 mid-flight.

In an interview, saying “rolling deploy requires forward and backward
compatible events” is the difference between a pretty diagram and a
system you could actually ship.

## Check yourself

1. New code adds a field. Old code reads the record and writes it back.
 What must the old code do to not destroy the new field?
2. Why are pickle files a bad choice for a queue between two services?
3. Server-first vs client-first: which compatibility direction is
 stricter for a mobile app?
4. A Temporal workflow runs for two weeks while you deploy daily. What
 can you change in the workflow payload, and what can you not?

Continue to [Replication](../6-replication/).

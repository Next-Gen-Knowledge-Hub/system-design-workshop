# 5. Encoding and evolution

These notes follow Chapter 5 of *Designing Data-Intensive Applications*, 2nd edition, by Kleppmann and Riccomini.

A record in memory is not the thing another program receives. The thing on the wire or on disk is bytes, and the program that decodes those bytes is often a different version from the program that wrote them. This chapter is how a system changes without stopping: a rolling upgrade, a schema that old and new code can both read, and the path the bytes travel, because the path decides which version is allowed to be old.

## Rolling upgrades

**Maintainability** at this level is a property of the bytes. A **rolling upgrade** starts new code on a few nodes while old code is still running on the others. There is no **flag day**, no moment when every writer and every reader must switch together. Two directions have to hold at once.

**Backward compatibility**: new code can read data that old code wrote. **Forward compatibility**: old code can read, or at least tolerate, data that new code wrote. The usual tolerance is to ignore fields the old code does not know.

The failure mode is quiet. New code writes a field. Old code decodes the record into a model object that only has the fields it was compiled with, and that object does not keep the unknown bytes. Old code writes the record back. The new field is gone. Nobody threw an error. The loss is a missing key on the next read.

Every encoding format either gives you a way to do both directions on purpose, or it leaves you to discover this strip-on-write in production.

## Language-specific encodings

Java serialization, Python pickle, and Ruby Marshal turn an in-memory object into bytes with almost no ceremony. They are bound to one language. They have a history of decoding as code execution: an untrusted payload is not data, it is a program that runs inside the reader, a remote-code-execution hole. They are weak at compatibility, because the bytes assume the class layout of the writer, and a newer or older class does not have a story for extra or missing fields.

A queue between two services is a bad place for them. The queue holds the bytes until some other process reads them, so it is both a wire and a long-lived file, and the reader may be another language, another version, or a curious human who should not be able to run the producer's objects. Keep these encodings inside one process. Do not put them on a public wire or in a file that outlives the deploy that wrote it.

What you lose: a pickle written by today's service is a code-execution bug for anyone who loads it, and it is unreadable to a reader that is not that Python, that class, that version.

## JSON, XML, CSV, and the binary cousins

JSON, XML, and CSV are everywhere, readable by a person, and optional about schemas. JSON is the default body of an HTTP API. The costs are concrete. The text is verbose next to a binary encoding of the same fields. Numbers are ambiguous: an integer, a float, and a 64-bit id are all "a number," and JavaScript's number cannot hold every 64-bit integer, so an id can round-trip into a different value. There is no native binary type; bytes become a string of some encoding you agreed on the side. CSV adds quoting: a comma or a newline inside a field changes the shape of the row unless every writer and reader uses the same escape rules.

These formats were not designed to be backward or forward compatible. They have no field numbers and no step that resolves a writer schema against a reader schema. A typical decoder binds known keys onto a struct and drops the rest. You also cannot reuse a field for a new meaning. The key `status` is just a name. If old writers stored `1` for "active" and new writers store `"paused"` in the same key, every reader sees one key and has to guess which era it is looking at. Nothing in the format distinguishes the two meanings.

Compatibility, when you have it, is a convention you enforce yourself. Add fields. Never reuse a key for a new meaning. Be liberal about unknown keys: if you might write the record back, keep the keys you did not understand.

A worked round trip shows the drop. The stored document is `{"id": 17, "name": "Sara"}`. A new writer adds a city and stores `{"id": 17, "name": "Sara", "city": "Tehran"}`. An old process loads that JSON into a struct of `id` and `name`. The struct has nowhere to put `city`, so the city is discarded. The old process saves the user and writes `{"id": 17, "name": "Sara"}`. The next reader, new code included, sees no city. The field was lost on a write that thought it was only updating the name.

MessagePack, BSON, and CBOR are the binary cousins. They are smaller on the wire. They are still schemaless, so they have the same evolution limits: unknown fields survive only if your code keeps them, and a reused key still has no version of its own.

What you lose: a 64-bit id that passes through JavaScript and comes back rounded; a CSV row split on a comma that was data; a city field stripped by an old reader that wrote the document back; a `status` key that means two different things to two deploys.

## Protobuf, Thrift, and Avro

Protocol Buffers, Thrift, and Avro are binary formats driven by a schema. The schema is documentation a compiler can check. It is also the input to code generation, so writers and readers in several languages share one field list. The cost is that you cannot `cat` a Protobuf file and read it. You need tooling. For any bytes that will live longer than a short-lived hack, that trade is worth making, because compatibility becomes a review comment — "this change is backward compatible" — instead of a broken reader at 03:00.

Protobuf and Thrift identify fields by number.

```
field 1: id = 17
field 2: name = "Sara"
field 3: city = "Tehran"   (added later, optional)
```

A new reader facing old bytes, which have no field 3, fills the default for city. That is backward compatibility: new code reads old data, and it can modify that data when it writes again. An old reader facing new bytes sees field number 3, does not know that number, and skips it. The old code does not interpret the new field and does not assign it a different meaning. That is the forward-compatible direction, and it is why these formats are built for both directions. Do not change the type of a field number in place: field 2 as a string and field 2 as an integer are not the same field, and old and new readers will both think they understand it.

Skipping a field number is not the same as keeping it across a write. If the old generated object discards unknown fields instead of retaining the raw bytes, the save that follows is the same strip as the JSON example: city was on the wire, and it is not in the bytes written back. "Old versions discard the new field" is safe only when discard means "do not modify it, leave it in the record." Generated code in many languages exists for both Protobuf and Thrift. gRPC rides on Protobuf, so an internal RPC using gRPC inherits this field-number story.

Avro puts the schema on the write. A **schema registry** stores each schema version under an id, which is the usual way a payload says "I was written with schema 14" without embedding the whole schema every time. The reader's schema and the writer's schema are **resolved** against each other: fields are matched, and the rules of that match are the compatibility story. A field added with a default can be filled when an old writer omitted it. A field removed from the reader schema can be ignored. This is the encoding most at home in Hadoop and Kafka pipelines, where the consumer that reads a record may have been deployed on a different day from the producer that wrote it.

What you lose if you renumber or retype a field: old and new code silently disagree about what field 2 is. What you lose if unknown field numbers are dropped on serialize: the same data loss as the JSON round trip, with a format that was capable of avoiding it. What you lose without a registry for Avro: the reader guesses which schema wrote the bytes, and the resolution step has nothing to resolve.

## Through a database

Encoding matters when data moves, and a database is a move with a long delay. Process A writes a row. Process B reads it later, maybe years later, on new code. The database is the integration point between two deploys that will never be in the same room.

The strip-on-write bug is the classic one. Process B loads JSON, drops keys its model does not know, and writes the row back. Fields a newer producer added are gone. An ORM that maps only the columns it knows does the same thing to a table: the new column is not on the object, so writing that object back does not keep the column a newer writer added. Unknown fields have to survive a read-modify-write. The old code does not have to understand them. It has to put them back.

What you lose: a column added this week, wiped by a worker from last year that "only updates the name," running against a row that will still be here in two years.

## Through services: REST and RPC

A request through a service is a shorter trip. The client encodes a request, the server decodes it, the server encodes a response, the client decodes that. Both ends are deploys you can often order.

REST with JSON is simple, and intermediaries can cache a response whose URL and headers say it is cacheable. RPC — gRPC, the older SOAP stacks, Finagle, and others — presents the same trip as a function call in the client code. The network stays hidden until a timeout. A timeout is not "the function did not run." The request may have been applied on the server, and only the response was lost. A retry can apply it a second time. The call is a network hop with partial failure, even when the source code looks local.

The compatibility direction is kinder than a database if you control the deploy order. Upgrade servers first, then clients. Requests need to be backward compatible: a new server must read an old client's request. Responses need to be forward compatible: an old client must tolerate a new server's response, usually by ignoring new fields. Mobile clients break the assumption. An old app binary stays installed for years. You cannot finish the rollout by waiting for those clients. Responses have to stay forward compatible for as long as the oldest app in the wild still calls you. That is the stricter direction for a mobile API.

A public HTTP API from this chapter's rules: JSON, a version in the URL or in fields you only add, and a field's meaning never mutated in place. An internal high-QPS service: gRPC and Protobuf, servers upgraded before clients.

What you lose if you upgrade clients first: the new client sends a field the old server rejects or misreads. What you lose if a mobile response changes a field's meaning: the old app, still installed, does the wrong thing for years. What you lose if you treat a timeout as "nothing happened": the retry doubles a charge that actually succeeded.

## Workflows that outlive a deploy

Some business processes are longer than one RPC. Onboard a merchant. Settle a payout. Retry a KYC (know-your-customer) check for three days. A crash in the middle must not forget that the card was already charged.

**Workflow engines** such as Temporal and Cadence persist each step as encoded state. The process can die, another machine can load that state, and the workflow continues from the step that was recorded. The state is ordinary encoded data, except it has to remain readable across code deploys while the workflow is still open. A payout workflow can stay open for two weeks while you deploy daily. Today's code will load a payload written on day one.

Treat the workflow input and each step's result with the same rules as an event schema. You can add an optional field that old code knows how to skip. You cannot rename a field in the middle of an open workflow, and you cannot change what a field means, because a step recorded yesterday will be replayed by today's code. Version the workflow definition, so an in-flight run stays on the definition it started with. A homegrown state machine stored in Redis has the same encoded payload and the same multi-day lifetime, and it has none of the engine's tooling for replay and versioning. The problem does not arrive with the product name. It arrives with "we saved the step."

What you lose if you rename `amount` to `amount_cents` during those two weeks: runs that started under the old name fail or, worse, read the wrong field and move a decimal point. What you lose if the step is only in memory: the crash forgets that the card was charged, and the retry charges it again.

## Events and actors

In an event-driven path, a producer writes a message and a consumer reads it later, on its own schedule. The pipe is a message broker, or a **log**: an append-only ordered sequence of records that each consumer reads at its own position. Compatibility is stricter than in RPC. You cannot assume the consumer was upgraded first, or at all, because it is a different deploy, and it may be reading a record that was written last week.

A schema registry plus Avro or Protobuf exists for this path. The check belongs in CI: a proposed schema change is rejected when it is not compatible with the schemas already in the registry, before the producer ships. If a consumer republishes a record to another topic, it has to forward fields it does not know. Otherwise it becomes the strip-on-write bug again, one hop downstream, and every later consumer inherits the stripped record.

**Actor** systems — Akka, Orleans, Erlang — are message passing with **location transparency**: the sender uses the same message whether the recipient is in this process or on another machine. The messages still need an encoding that survives a rolling upgrade, for the same reason any other message does. The model already allows the network to drop a message, so the sender treats loss as a case it handles, not as a surprise. An RPC client that pretends the call was local discovers that loss as a timeout it did not structure the code for. Actors fit distribution better because the drop was part of the programming model from the start. They do not remove the need for a compatible encoding.

What you lose if a consumer strips unknown fields and republishes: the new field dies at the first old consumer, and the rest of the pipeline never sees it. What you lose if the registry check is skipped: the incompatible schema is discovered by a consumer that cannot decode Tuesday's records.

## Recap

A rolling upgrade needs both directions at once. New code reads old data. Old code tolerates new data, which means ignoring unknown fields and, on any write-back, putting those fields back. A model object that drops unknown fields deletes them the next time it saves.

Pickle, Java serialization, and Ruby Marshal are one language, a remote-code-execution risk on decode, and a weak compatibility story. They do not belong on a public wire, in a long-lived file, or on a queue between services.

JSON, XML, and CSV were not built as compatible evolution formats. They are verbose, ambiguous about numbers (including 64-bit ids that JavaScript cannot hold), and, for CSV, easy to split on a character that was data. Decoders drop unknown keys. A key cannot be reused for a new meaning. The workable convention is: add keys, keep their meanings, preserve keys you do not understand. MessagePack, BSON, and CBOR are smaller and still schemaless.

Protobuf and Thrift use field numbers. Old readers skip unknown numbers and must not rewrite them away. New readers read old data and can modify it. Changing a type in place breaks both directions. gRPC uses Protobuf. Avro resolves a writer schema against a reader schema, usually via a schema registry, which is the compatibility mechanism in Hadoop and Kafka. Schemas buy a compiler check and a reviewable compatibility rule, at the cost of tooling to inspect the bytes.

A database holds the bytes for years, so an old reader that writes a row back can strip a new column, including through an ORM that only maps known columns. REST plus JSON is simple and cacheable. RPC looks like a local call until a timeout, and the timeout may mean the server did the work. Upgrade servers first: backward compatible requests, forward compatible responses. Mobile clients linger for years, so response compatibility is the direction you cannot retire.

A workflow engine such as Temporal or Cadence persists each step so a crash does not forget a charge. The payload must survive daily deploys while a run stays open for days or weeks, including a three-day KYC retry and a two-week payout. Add optional fields; do not rename or redefine a field mid-flight; version the definition. A Redis state machine has the same constraint without the tooling.

Events are read later by consumers you do not upgrade in lockstep, so a registry and a CI compatibility check carry Avro or Protobuf. Republish only if unknown fields are forwarded. Actors (Akka, Orleans, Erlang) pass messages with location transparency and already expect the network to drop one; the encoding still has to roll forward with the code.

The matching choices: a public HTTP API in JSON, versioned in the URL or by fields you only add; internal high-QPS calls in gRPC and Protobuf, servers first; Kafka pipelines in Avro or Protobuf with a registry and a CI check; mobile responses that stay forward compatible; workflow payloads that are versioned and never renamed while a run is open.

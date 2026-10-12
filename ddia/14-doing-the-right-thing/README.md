# Doing the right thing

These notes teach the responsibilities that come with data-intensive systems from *Designing Data-Intensive Applications*, 2nd edition, by Martin Kleppmann and Chris Riccomini.

## What this subject is

The last part of the material is not an appendix. The same logs, profiles, and models that scale can harm people. A complete design says **who is in the data** and **what the system is allowed to do to them**. Predictive systems, surveillance, power, and law are engineering constraints. They sit on the same spine as storage and recovery.

Three constraints show up as soon as a system stores people. **Data minimization**: collect and keep only what the purpose needs. **Purpose limitation**: data collected for one reason is not silently reused for another. **Deletion versus immutability**: a replayable log wants to keep every event forever, and a person can have a right to be removed from it. The GDPR, the European Union's general data-protection regulation, is one law that makes those constraints enforceable, including a right to erasure. Other laws do similar work. The rest of this lesson is what those constraints look like inside a pipeline.

You can build a recommendations pipeline that is technically impressive — similarity vectors, a training job, live features — and that is unfair, illegal, or impossible to appeal. Designs sketched for an interview rarely mention this. Production systems get fined, and they hurt people. Treat the questions below as part of the design, not as extra credit.

## Scoring people

Models score people. Credit. Hiring. Fraud. Feed ranking. Content moderation. Policing. The score is **derived data** with teeth: a value computed from stored history, then used as an automatic decision at scale.

Walk one loan application through the pipeline. The application arrives. A batch job once trained a model on past lending decisions. A feature job assembles this applicant's zip code, school, and device. The model emits a score. A threshold, chosen by someone, turns that score into "deny." The applicant is rejected in milliseconds. Everything after this section is about that path: where the "no" came from, who owns it, and what happens when it is wrong.

### The training set replays the past

Historical data contains historical injustice. A model that "just predicts the labels" will replay them. If past loan officers denied a group, and those denials are the labels, the model learns to deny that group. Dropping a `race` column or a `gender` column does not remove the signal. **Proxies** leak the protected attribute anyway: zip code, name, school, the price of the device. Accuracy on a test set is a number about the labels you already have. It is not fairness across groups. A model can be right about the historical labels and systematically wrong about the people.

If you cannot explain why this applicant was denied, or the process forbids the explanation, two things fail at once. The applicant cannot appeal. You cannot debug. Explainability and an audit log are part of the **architecture**. They are not a PDF someone writes for legal after the pipeline is already in production. The audit log has to be able to reproduce the inputs, the model version, and the threshold that produced this score.

### A person shipped the threshold

"The algorithm decided" is not a sentence that names an owner. Someone shipped the pipeline. Someone chose the training window. Someone chose the threshold. Someone chose not to build an appeal path. Operability includes a bad model the same way it includes a bad deploy. Who is on call. How you roll a threshold back. How you reproduce a score when a complaint arrives.

High-impact actions need a **human process** and a paper trail. Freezing an account, denying a loan, and shadowbanning a person are in this class. You would not let a cron job drop production tables with no safeguard. Do not let a score do the equivalent to a person with no safeguard either.

A fraud model can be very accurate and still be a design failure. Accuracy counts the cases the model gets right. A false positive that freezes a person's account, with no appeal and no way to reproduce the score, is harm. Treat that false-positive path as a first-class target, in the same list as p99 latency. The latency number does not excuse a freeze nobody can undo.

### A loop that confirms itself

A closed loop feeds its own outputs back in as training data.

On a "for you" timeline the loop is concrete. The ranker puts a post high. The post gets the click because it was put high. The click is a positive label. The next training run trusts that label. Polarizing or extreme posts draw clicks, so they win the next round, so they are shown more, so they draw more clicks. Engagement as the only metric is the loop with no brake.

A fraud model that freezes accounts and never builds a path back does two kinds of damage. The person is stuck. Support drowns in tickets that the model will never see as corrections, so the model never learns which freezes were wrong.

Predictive policing is the same shape with arrests as the label. The model sends patrols where it expects crime. More patrols produce more arrests there. The arrests become the next training set. The model "learns" that it was right about that neighborhood, because it is reading its own deployment.

When you design a closed loop, design a **governor** beside it.

- Diversity constraints, so one kind of item cannot occupy the whole slate.
- A limit on exploitation versus exploration, so the system sometimes shows something the current ranker would not have picked, and can learn from that.
- Rate limits, so one feedback spike cannot rewrite the model overnight.
- A human review on high-impact actions, before or immediately after they land.
- Metrics that are not only engagement. Appeals upheld, freezes reversed, and complaints reproduced are metrics.

## Privacy and tracking

### Retention became a policy

Surveillance is not only a state activity. Advertising, analytics SDKs, employer telemetry, session replay, and "we keep every click forever because the lake is cheap" are surveillance infrastructure. They record a person's behavior at a resolution the person did not meaningfully choose.

Cheap object storage removed the old natural limit. Disk used to be expensive, so systems deleted. A cheap lake made keeping the bytes the default. Retention is now a **policy choice**, not an accident of hardware. "We might want it" is not a retention period.

### Consent that is not a choice

A 40-page modal, a pre-checked box, and "accept all to use the product" are not freedom of choice. The person cannot refuse and still use what they came for, or the refuse button is hidden. Those are dark patterns. They are product decisions with a data system behind them. The system is able to store less. The system is able to make reject-all as easy as accept-all. Engineering is what makes either one real. A consent flag that the pipeline ignores, while the lake keeps every event anyway, is the dark pattern implemented in storage.

### Use the data for the reason you collected it

**Purpose limitation** is the rule that billing data stays billing data. Take a card number and a shipping address collected to charge for an order, and feed them into a personality model. That secondary use fails the GDPR, and it fails the person's trust, because the person agreed to checkout, not to a psychological profile. The secondary use needs a new consent, or it needs to not happen.

The design moves that enforce this are ordinary systems work. Access control, so the personality-model job cannot read the billing store. Separate stores, so the two purposes are not one table. **Tokenization**, replacing the raw identifier with a token that only the billing system can turn back into a card number. And a refusal to pipe personally identifying information into the warehouse "just in case." Derived pipelines help when they are built so that search and analytics receive a **redacted** copy by default: the order id and the country, not the email and the card. The same pipelines hurt when they do the opposite and fan the raw identifier out to every index, every feature set, and every log.

### A copy is a form of power

Who can query the lake? A derived "user 360" view joins every store into one row per person. It is useful for the product. It is equally useful to an attacker who steals it, and to a government request that asks for it. Minimize the copies. Encrypt them. Audit who reads them. Do not put email addresses in the search index if search does not need email addresses. Concentration of data is concentration of power. The architecture either spreads that concentration or intensifies it. One joined view that every internal tool can query is the intensified form.

### Norms, while they are still being set

The analogy is the industrial revolution. Societies were early in writing factory safety and labor law while the machines were already running. Data-intensive systems are early in the same way. Waiting for a perfect law is not a plan. Ignoring the norms that are already emerging is not a plan either. Engineers ship the infrastructure that makes abuse easy or hard: the retention default, the redact-on-derive step, the appeal queue, the access audit.

**Legislation** — the GDPR, the CCPA, sector rules, the EU AI Act and its cousins — and **self-regulation** — PCI, SOC 2, an internal review board — are incomplete. They are still the constraints you design under. Ignoring them makes the system wrong in the same way that an SLA you cannot meet makes the system wrong. The promise is part of the spec, whether or not you wrote it down.

## Erasure against a log that wants to keep everything

"Delete me" against a replayable log is an engineering problem. The log's value is that you can reprocess it. The person's right is that reprocessing must not resurrect them.

| Technique | What you do | What goes wrong |
|---|---|---|
| Crypto-shredding | Encrypt that person's bytes under a key that belongs to them, then delete the key. The ciphertext is still on disk and is junk. | You have to manage the keys. A backup of the key, or a copy of the key you forgot, decrypts the person again. |
| Tombstones and suppress lists | Write a marker that says "skip or redact this person." Every reprocess reads the marker. | The marker has to reach every derived store. One job that does not load the list brings the person back. |
| Filter-on-read | Leave the bytes where they are. Refuse to serve them. | Any consumer that forgets the filter serves the person. The bytes are still in the file a stolen backup contains. |
| Rewrite and reprocess | Physically purge the person from the lake partitions. Write a new snapshot that does not contain them. | It is expensive. It depends on the discipline of writing a new immutable snapshot rather than editing the only copy in place. |

Pick one on purpose. Then list every copy, because a delete that only removes the OLTP row is fiction. The email address in a typical system is in at least these places:

- the OLTP row
- the Kafka topic (and any compacted changelog that still holds the latest value)
- the Elasticsearch document
- the warehouse or lake files
- backups of all of the above
- model training sets built from those files
- application logs that printed the address

"Delete me" means a different operation on each. The OLTP row can be deleted or overwritten in place. The topic cannot: an append-only log does not un-write an offset, so you tombstone, crypto-shred, or rewrite and cut a new snapshot. The search document has to be deleted or replaced, or the next query returns the address. The warehouse file is immutable, so you write a new snapshot of the partition and move readers to that snapshot. A backup restores the old bytes unless the backup expires or the key that decrypts it is gone. A training set baked into a model is still there after the lake is clean; the model is another copy, and it needs a rebuild that excludes the person. A log line on a debug volume is a copy too.

Derived pipelines are how this either works or fails at scale. If every derived store is a function of the record, a redact or a suppress list in that function reaches search, the warehouse, and the features on the next replay. If each store was filled by its own dual write, you have to find every write and hope you remember them all.

## What the design has to say

A complete design states the personal-data facts in the same place it states the data flow.

- What personal data is stored.
- Where the copies are: OLTP, the log, search, the warehouse, backups, training sets, application logs.
- How long each copy is retained, as a policy, not as "the disk had room."
- Who can query it.
- How deletion propagates to logs, backups, streams, and models.
- Which decisions are automated, and what the appeal path is: who is on call, how the threshold rolls back, how the score is reproduced for a complaint.

If the product is a feed, the design names the feedback loop and the governor. If it is credit or fraud, false positives are a target in their own right, because they are user harm. If it is ads or tracking, the design names minimization and purpose: what was collected, for which reason, and which pipeline is forbidden to read it.

## Recap

A model that scores people is derived data used as a decision. It will replay whatever injustice is in the labels, including through proxies, and accuracy on a test set does not answer fairness or appeal. Someone owns the training window, the threshold, the rollback, and the paper trail; a freeze or a denial without that path is a broken design even when the model is accurate. Closed loops — a feed, a fraud freeze, predictive policing — confirm themselves unless a governor limits them. Cheap storage turned retention into a policy. Purpose limitation, redacted derived copies, and a chosen erasure technique (crypto-shred, tombstone, filter-on-read, or a rewritten snapshot) are what make "delete me" true for every copy, not only for the OLTP row.

# 14. Doing the right thing

Companion notes for **Chapter 14** of *Designing Data-Intensive Applications*
(2nd ed.). The last chapter is not an appendix. The same logs, profiles,
and models this book teaches you to scale can harm people. A complete
system design includes **who is in the data** and **what the system is
allowed to do to them**.

Chapter 1 already raised GDPR, deletion vs immutability, and data
minimization. This chapter zooms out to predictive systems, surveillance,
power, and law — still as engineering constraints, not as a TED talk.

## What this chapter is for

You can design a technically beautiful recommendations pipeline (ch. 4
vectors, ch. 11 training, ch. 12 features) that is unfair, illegal, or
impossible to appeal. Interview designs rarely mention this. Production
ones get fined — and hurt people. Treat this folder as part of the
spine, not extra credit.

## Predictive analytics

Models score people: credit, hiring, fraud, feed ranking, content
moderation, policing. They are **derived data** (ch. 13) with teeth —
automated decisions at scale.

### Bias and discrimination

Historical data contains historical injustice. A model that “just
predicts the labels” will replay them. Even without a `race` or
`gender` column, **proxies** (zip code, name, school, device price)
leak protected attributes. Accuracy on a test set does not mean fairness
across groups.

If you cannot explain *why* someone was denied (or your process forbids
it), they cannot appeal and you cannot debug. Explainability and audit
logs are part of the **architecture**, not a PDF for legal later.

### Responsibility and accountability

“The algorithm decided” is not a grown-up sentence. Someone shipped the
pipeline, chose the training window, chose the threshold, chose not to
build an appeal path. Operability (ch. 2) includes: who is on call for
a bad model, how you roll back a threshold, how you reproduce a score
for a complaint.

High-impact actions (freeze account, deny loan, shadowban) need a
**human process** and a paper trail — the same way you would not let a
cron job drop production tables without safeguards.

### Feedback loops

The feed ranks what gets clicks; clicks train the ranker; polarizing
or extreme content can win. Fraud models that freeze accounts without
a path back create support hell *and* harm. Predictive policing sends
patrols where the model expects crime; more arrests there; the model
“learns” it was right.

When you design a closed loop, design a **governor**: diversity
constraints, exploration vs exploitation limits, rate limits, human
review on high-impact actions, metrics that are not only engagement.

## Privacy and tracking

### Surveillance

Surveillance is not only states. Ads, analytics SDKs, employer
telemetry, session replay, and “we keep click logs forever because the
lake is cheap” are surveillance infrastructure. Cheap object storage
(ch. 1 / 11) removed the old natural limit (disk was expensive, so we
deleted). Now retention is a **policy choice**, not a hardware
accident.

### Consent and freedom of choice

Consent that is a 40-page modal, a pre-checked box, or “accept all to
use the product” is not freedom of choice. Dark patterns are product
decisions with a data-system backing: you *can* store less; you *can*
make reject-all as easy as accept-all. Engineering enables both.

### Privacy and use of data

**Purpose limitation** (ch. 1): collected for billing, reused for a
personality model, is how you fail GDPR *and* user trust. Access
control, separate stores, tokenization, and not piping PII into the
warehouse “just in case” are design. Derived pipelines (ch. 13) help
when search indexes and analytics get **redacted** copies by default.

### Data as assets and power

Who can query the lake? A derived “user 360” view is a gift to both
the product and an attacker — or a government request. Minimize copies;
encrypt; audit access; do not put emails in the search index if search
does not need emails. Concentration of data is concentration of power;
architecture either spreads or intensifies that.

### Remembering the industrial revolution

The book’s analogy: we are early in norm-setting for data-intensive
systems, the way industrial societies were early on factory safety and
labor law. Waiting for perfect law is not a plan; neither is ignoring
emerging norms. Engineers ship the infrastructure that makes abuse easy
or hard.

### Legislation and self-regulation

**Legislation** (GDPR, CCPA, sector rules, EU AI Act and cousins) and
**self-regulation** (PCI, SOC 2, internal review boards) are incomplete;
they are still the constraints you design under. Ignore them and the
system is wrong the same way an SLA you cannot meet is wrong.

**Right to erasure vs replayable logs** is an engineering problem, not
a vibe:

| Technique | Idea | Trade-off |
|---|---|---|
| Crypto-shredding | Delete the key; ciphertext becomes junk | Key management; backups of keys |
| Tombstones / suppress lists | Reprocessing skips or redacts the user | Must reach every derived store |
| Filter-on-read | Leave bytes; never serve them | Risky if any consumer forgets |
| Rewrite / reprocess | Physically purge from lake partitions | Expensive; needs immutable→new snapshot discipline |

Pick one on purpose; list every copy (OLTP, Kafka, Elastic, warehouse,
backups, model training sets, logs). “Delete me” that only hits the
OLTP row is fiction.

## How this shows up when you design something

Add to the ch. 13 checklist:

6. What personal data is stored, where the copies are, retention, who
   can query it, how deletion propagates to logs, backups, streams, and
   models, and which decisions are automated with what appeal path.

If the design is a feed, mention feedback loops. If it is credit/fraud,
treat **false positives** as a first-class SLO (user harm), not only
p99 latency. If it is ads/tracking, mention minimization and purpose.

## Check yourself

1. List every copy of an email address in a “typical” system that has
   OLTP + Kafka + Elastic + warehouse + backups. What does “delete me”
   mean for each?
2. Why is a very accurate fraud model still a design failure if false
   positives have no appeal?
3. How does a feedback loop show up in a “for you” timeline?
4. Name one technique to honor erasure without throwing away the entire
   event log — and one failure mode of that technique.
5. Purpose limitation: give an example of a secondary use that should
   require a new consent or a new store.
6. Why did cheap lakes make retention an ethics problem instead of a
   disk problem?
7. Who is accountable when a model denies a loan — in an architecture
   doc, not a press release?
8. How do derived-data pipelines (ch. 13) help privacy if you design
   them that way — and how do they hurt if you do not?

This is the last DDIA chapter in the workshop.

Next (later): [Interview-style system designs](../15-interview-designs/)
and [Further reading](../16-further-reading/).
The [trade-offs cheat sheet](../DESIGN_TRADEOFFS.md) is there when you
want the whole book on one screen.

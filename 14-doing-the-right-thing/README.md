# 14. Doing the right thing

Companion notes for **Chapter 14** of *Designing Data-Intensive Applications*
(2nd ed.). The last chapter is not an appendix. The same logs, profiles,
and models this book teaches you to scale can harm people. A complete
system design includes **who is in the data** and **what the system is
allowed to do to them**.

Chapter 1 already raised GDPR, deletion vs immutability, and data
minimization. This chapter zooms out to predictive systems, surveillance,
and law.

## What this chapter is for

You can design a technically beautiful recommendations pipeline (ch. 4
vectors, ch. 11 training, ch. 12 features) that is unfair, illegal, or
impossible to appeal. Interview designs rarely mention this. Production
ones get fined. Treat this folder as part of the spine, not extra
credit.

## Predictive analytics

Models score people: credit, hiring, fraud, feed ranking, policing.
They are **derived data** (ch. 13) with teeth.

**Bias and discrimination:** historical data contains historical
injustice. A model that “just predicts the labels” will replay them.
Even without a `race` column, proxies (zip code, name, school) leak
through. If you cannot explain *why* someone was denied, you cannot
fix it and they cannot appeal.

**Responsibility:** “the algorithm decided” is not a grown-up sentence.
Someone shipped the pipeline, chose the training window, chose the
threshold. Logging, audit trails, and a human process for exceptions
are part of the **architecture** (operability, ch. 2).

**Feedback loops:** the feed ranks what gets clicks; clicks train the
ranker; polarizing content wins. Fraud models that freeze accounts
without a path back create support hell *and* harm. When you design
a closed loop, design a **governor** (diversity, rate limits, human
review on high-impact actions).

## Privacy and tracking

**Surveillance** is not only states. Ads, analytics SDKs, employer
telemetry, and “we keep click logs forever because the lake is cheap”
are surveillance infrastructure. Cheap object storage (ch. 1 / 11)
removed the old natural limit (disk was expensive so we deleted).

**Consent** that is a 40-page modal is not freedom of choice. Dark
patterns are a product decision with a data-system backing (you *can*
store less).

**Use of data:** purpose limitation (ch. 1) — collected for billing,
reused for a personality model, is how you fail GDPR *and* user trust.
Access control, separate stores, and not piping PII into the warehouse
“just in case” are design.

**Data as power:** who can query the lake? A derived “user 360” view
is a gift to both the product and an attacker. Minimize copies (ch. 13
actually helps: if search should not have emails, do not put emails in
the index).

The book’s industrial-revolution analogy: we are still early in
norm-setting. **Legislation** (GDPR, CCPA, EU AI Act, sector rules)
and **self-regulation** (PCI, SOC 2) are incomplete; they are still
the constraints you design under. Right to erasure vs replayable logs
is an engineering problem: crypto-shredding, tombstones, filter-on-read,
reprocessing with a suppress list. Pick one on purpose.

## How this shows up when you design something

Add a fifth section to the ch. 13 checklist:

6. What personal data is stored, where the copies are, retention, who
   can query it, how deletion propagates to logs, backups, and models,
   and which decisions are automated with what appeal path.

If the design is a feed, mention feedback loops. If it is credit/fraud,
mention false positives as a first-class SLO, not only p99 latency.

## Ties to other workshops

- Kafka retention and compaction: deletion is not “the consumer acked.”
  You need a policy.
- Search indexes and caches are extra copies for GDPR.
- [k8s secrets](../../k8s-workshop/2-resources/secret/README.md) —
  necessary, not sufficient, for privacy.

## Check yourself

1. List every copy of an email address in a “typical” system that has
   OLTP + Kafka + Elastic + warehouse. What does “delete me” mean?
2. Why is a very accurate fraud model still a design failure if false
   positives have no appeal?
3. How does a feedback loop show up in a “for you” timeline?
4. Name one technique to honor erasure without throwing away the entire
   event log.

This is the last DDIA chapter in the workshop.

Next (later): [Interview-style system designs](../15-interview-designs/)
and [Further reading](../16-further-reading/).
The [trade-offs cheat sheet](../DESIGN_TRADEOFFS.md) is there when you
want the whole book on one screen.

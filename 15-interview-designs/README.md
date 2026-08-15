# 15. Interview-style system designs

**This section is a placeholder.** We are not filling it yet.

The spine of this workshop is *Designing Data-Intensive Applications*.
Parts A–E (folders 1–14) stay aligned with that book. This folder is
where we will later add **timed, interview-shaped walkthroughs**: pick
a product, state load numbers, draw the system of record vs derived
stores, and argue the trade-offs using the vocabulary from DDIA.

## What will land here

Likely sources (Track C in the reading queue — drills, not a theory
replacement):

- Alex Xu, *System Design Interview* Volume 1
- Alex Xu, *System Design Interview* Volume 2
- Our own designs (feeds, payments, ride matching, unique IDs, etc.)

Each future write-up should:

1. Restate requirements the way [ch. 2](../2-nonfunctional-requirements/)
   taught (load, percentiles, failure).
2. Pick models and stores with [ch. 3–5](../3-data-models/).
3. Say replication / sharding / isolation out loud ([ch. 6–8](../6-replication/)).
4. Mark what is linearizable vs eventual ([ch. 10](../10-consistency-and-consensus/)).
5. Show the dataflow for derived data ([ch. 11–13](../11-batch-processing/)).
6. Spend one paragraph on data about people ([ch. 14](../14-doing-the-right-thing/)).

Until those notes exist, practice by taking any product you know and
answering the **Check yourself** questions in chapters 1–14, then
sketching it on paper with the [cheat sheet](../DESIGN_TRADEOFFS.md).

## Status

| Item | Status |
|---|---|
| DDIA companion (ch. 1–14) | Current |
| Xu Vol. 1 walkthroughs | Not started |
| Xu Vol. 2 walkthroughs | Not started |
| Original designs | Not started |

Continue to [Further reading](../16-further-reading/) for the other
books we will fold in later.

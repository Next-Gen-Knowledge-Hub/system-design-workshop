# 16. Further reading in this field

DDIA is the **data-systems spine** ([`../ddia/`](../ddia/), chapters
1–14). *Software Architecture: The Hard Parts* is now a **second track**,
not a stub: [`../hard-parts/`](../hard-parts/). Do not merge those
chapters into DDIA folders; use [`../INDEX.md`](../INDEX.md) when a word
appears in both.

This page is for books we have **not** turned into tracks yet.

## Planned, not written

| Book | Why it belongs here | Status |
|---|---|---|
| *Understanding Distributed Systems* — Roberto Vitillo | Shorter pass over the ch. 9–10 terrain | Later |
| *System Design Interview* Vol. 1 & 2 — Alex Xu | Timed drills; lives mainly in [section 15](../15-interview-designs/) | Later |
| *Designing Distributed Systems* — Brendan Burns | Container-era patterns for placing and composing services | Later |
| *Software Architecture: The Hard Parts* — Ford, Richards, Sadalage, Dehghani | Evolutionary architecture, coupling, ownership, sagas | **Track B** — [`../hard-parts/`](../hard-parts/) |

## How new books will be added

When we read the next book:

1. Keep DDIA under `ddia/` (do not renumber chapters 1–14, do not
   hoist them back to the repo root).
2. Keep Hard Parts under `hard-parts/` (do not fold it into `ddia/`).
3. Either extend [section 15](../15-interview-designs/) (if it is a
   *design problem*) or add another track folder (if it is a *new lens*).
4. Cross-link with **See also**; do not merge chapters that share a word.

Until then, pick a path in [`../INDEX.md`](../INDEX.md): **DDIA 1 → 14**,
or **Hard Parts 1 → 15**, or one topic through both tracks.

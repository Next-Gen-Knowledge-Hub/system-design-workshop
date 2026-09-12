# Hard Parts track

Companion notes for **Neal Ford, Mark Richards, Pramod Sadalage, and
Zhamak Dehghani, *Software Architecture: The Hard Parts*** (O'Reilly,
October 2021). Subtitle: *Modern trade-off analysis for distributed
architectures*. This track stays on **how you pull a system apart and
put communication, data, and workflow back together** — coupling,
granularity, reuse, ownership, sagas, contracts, analytical mesh.

The other book in this workshop — *Designing Data-Intensive
Applications* (2e) — lives in [`../ddia/`](../ddia/)
([`../ddia/1-architecture-tradeoffs/`](../ddia/1-architecture-tradeoffs/) through
[`../ddia/14-doing-the-right-thing/`](../ddia/14-doing-the-right-thing/)). Shared
words (transaction, consistency, encoding, OLTP/OLAP, workflow) are
**not** merged here. Use [`../INDEX.md`](../INDEX.md) when you need the
other angle.

The book's running case is a **ticketing / field-ops company** ("Sysops
Squad"). These notes keep the *decisions*, not the plot.

| Folder | HP ch. |
|---|---|
| [1 No best practices](./1-no-best-practices/) | 1 |
| [2 Coupling](./2-coupling/) | 2 |
| [3 Modularity](./3-modularity/) | 3 |
| [4 Decomposition](./4-decomposition/) | 4 |
| [5 Component patterns](./5-component-patterns/) | 5 |
| [6 Operational data](./6-operational-data/) | 6 |
| [7 Service granularity](./7-service-granularity/) | 7 |
| [8 Reuse patterns](./8-reuse-patterns/) | 8 |
| [9 Data ownership](./9-data-ownership/) | 9 |
| [10 Distributed data access](./10-distributed-data-access/) | 10 |
| [11 Distributed workflows](./11-distributed-workflows/) | 11 |
| [12 Transactional sagas](./12-transactional-sagas/) | 12 |
| [13 Contracts](./13-contracts/) | 13 |
| [14 Analytical data](./14-analytical-data/) | 14 |
| [15 Build your own trade-off analysis](./15-trade-off-analysis/) | 15 |

# Architectural modularity

Software Architecture: The Hard Parts, Ford, Richards, Sadalage, Dehghani, O'Reilly 2021, Chapter 3.

## What a modularity driver is

A **quantum** is the smallest set of code and data you can deploy independently and still have it hang together. **Static coupling** is coupling you can see without running the system: imports, a shared schema, a shared library, a contract that changes in lockstep. **Dynamic coupling** is a runtime relationship between quanta: a call, a message, a wait. A **fitness function** is an automated or scheduled check that a characteristic you claimed still holds.

One honest quantum — a monolith, or a modular monolith whose modules still ship as one artifact — is a valid architecture. You pay for another seam only when a **driver** is real. A driver is a characteristic that is bad *because* too much lives in one quantum, and that would get better if a seam went in a specific place. "Better architecture" is not a driver. "Netflix does it" is not a driver. A driver points at a workflow, a metric, or a release log.

Five drivers are five arguments. Scaling assignment is a different claim from "we cannot test invoicing." Each split that wins on one driver loses somewhere else, usually in data and operations. The business case is that comparison.

The field-ops product has ticketing (intake, assignment, dispatch: a person, a slot, a site, with assignment latency and availability on the line) and nonticketing work (invoicing, tax, parts, knowledge articles, contracts). Same customers. Different cadence, scale curve, and failure domain.

## Maintainability

**Maintainability** is how safely a person can find and change the behavior they meant to change. In one quantum that holds two cadences, a tax-table edit and a dispatch-rule edit compete for the same attention, the same types, and the same review queue.

Every change becomes a search. A new hire cannot tell whether `Status` means ticket status, invoice status, or technician shift status. Touching billing recompiles assignment and puts assignment at risk again.

Put the seam where change reasons diverge. Files, types, and tests for one reason live together and do not mention the other reason. That seam can be packages inside one quantum. It does not have to be a service on the first day.

Crude measures still beat "the code feels messy": files touched per change-reason, time to a new teammate's first useful change on one workflow, and the number of cross-workflow imports in a diff.

A retailer with promotions logic copy-pasted through checkout, cart, and email spends every campaign week hunting several repositories. The driver is maintainability: one change, many homes. Scale is a different driver. A promotions module, still one quantum if the transaction has to stay together, is the matching first seam.

Splitting by technical layer — all controllers in one place, all repositories in another — makes navigation worse and leaves change reasons mixed. Extracting a service so small that one business change still hops five repositories misses the same driver.

## Testability

**Testability** is whether a change produces a fast, believable signal without standing up the whole system. One quantum that boots a database, a mailer, a search index, and last year's batch jobs before it can check tax rounding will be tested rarely and late.

The only honest test is then an end-to-end run on a full environment that is flaky and already booked. People stop writing tests for the part they changed. Regressions show up in production.

The seam lets you test a component or a domain with fakes at the boundary. The driver is "invoicing can be proven without dispatch running." Raising a coverage number by ten points is a different goal. A fitness function fits here: the pipeline fails if billing tests import dispatch, or if the billing test target pulls the whole monolith onto the classpath.

A hospital-billing module that needed a full patient-chart stack to assert a rounding rule, and that after the seam runs those rounding tests in seconds on a laptop, has bought the driver. The remaining end-to-end suite shrinks to contract tests at the seam.

If you cannot name a test you can run now and could not run before, the split did not buy testability. A fleet whose integration story is harder than the monolith's was spends the driver in reverse. Using production as the only place a change is tried is the same gap, left in place.

## Deployability

**Deployability** is how often and how safely a slice of behavior can go into production. It is the operational face of independent deploy: invoicing can miss this train without holding dispatch.

The pain is a release train. Ticketing bugfixes sit behind a finance freeze. Or every deploy is a full-system outage window, because nobody can tell what changed.

The seam makes cadence independent where the business already wanted that. Dispatch ships on Saturday. Tax tables ship on a weekday with extra review. Measure it in the release log: wait time per workflow, failed deploys whose cause was someone else's module, and how wide the rollback was.

A modular monolith still ships one artifact, so deployability moves only a little. Confidence can move more: smaller diffs, module test gates. Lockstep *releases* are an argument for another quantum, not only for another package.

A field-ops dispatch fix that waits three weeks because invoicing has an unfinished audit column is this driver. A split that leaves that column shared is the appearance of a seam.

Twenty pipelines that must still go green together, because of one shared migration, automate the train. They do not decouple it. Deployability is also the wrong banner for a shard whose actual pain was a missing test.

## Scalability

**Scalability**, as a modularity driver, means different parts want different amounts of hardware and are stuck sharing a process, a heap, or a connection pool. You scale the hot path and also scale the cold PDF renderer. Or you scale reads and also replicate a write-heavy ledger you did not want to replicate.

p99, and which load parameter is growing, are the evidence. A seam earns this driver when it lets you scale one capability without buying the same multiple for the others.

Assignment spikes when storms hit. Invoice PDF generation is a lunchtime batch. They share a JVM. Pods added for the storm pay for idle PDF capacity all year, or the batch knocks assignment over.

Cut where the scale curves differ. Sometimes the cut is a queue and a worker, still inside one network, rather than a public service. Sometimes it is a read replica used only for reporting. After the cut, check static coupling. Both sides left on one database primary are still tied together.

"We need to scale," with no load parameter, is incomplete. Dozens of services, each too chatty to scale on its own, move the bottleneck to the network and the orchestrator. A monolith that only needed more vertical capacity had a capacity ticket. It had no modularity driver.

A chat product whose search-indexing CPU dwarfs message ingest can split the indexer onto its own hardware, asynchronously, and that split matches this driver. Splitting "edit user avatar" into its own service matches it only when the numbers say so.

## Availability and fault tolerance

**Availability / fault tolerance**, as a driver, means a failure in one capability should not take down another. Today they share a fate because they share a process, a memory space, or a blocking call.

A leak in reporting exhausts memory and dispatch dies with the monolith. A hanging call to a tax API freezes every HTTP worker, including "create ticket."

Isolation is separate processes, separate connection pools, timeouts, and **bulkheads** (a separate pool, worker, or cgroup, so one capability cannot exhaust the resources of another). If the seam goes as far as another quantum, the survivor's critical path has no synchronous call into the part that is allowed to fail. The driver is failure containment. Evidence is an incident timeline whose blast radius crossed a workflow that did not need to be in that blast.

Dynamic coupling gives the isolation back. Two quanta plus a synchronous call on the hot path are one failure domain, with extra latency.

Three replicas of the same combined process survive a lost node. They do not survive a bad deploy or a poison query. Enough services, and a mesh outage becomes the shared fate the split was fleeing.

PDF generation of year-end statements pegs CPU and intake dies. A worker pool in its own cgroup is often the first availability seam, and it may still be one quantum. A new service is justified when the failure domain and the data can actually part.

## The business case

A **business case** is why the company spends money and risk on a driver now, and why this seam is the one to cut. "Migrate to microservices," with a capability map and with no cost of staying, no cost of moving, and no sequence, gives leadership a fashion proposal and gives engineers a long freeze on product work.

Write two futures, with the drivers as lines.

```
  stay (one quantum)                 move (named seams)
  ------------------                 ------------------
  deploy: 1 train / 3 weeks          dispatch: daily
  blast: reporting OOM kills all     reporting isolated
  test: 40 min full stack            billing unit tests 20s
  scale: storm => scale PDFs too     storm => scale assign only
  cost to stay: SLA credits,         cost to move: dual-write,
  heroics, hiring freeze on          lag, more on-call, slower
  "scary" modules                    first year of delivery
```

Keep the comparison specific.

- Cost the status quo in incidents and wait time. "Saturday dispatch was lost twice this year because of billing deploys" is a case. "Monoliths don't scale" is a slogan.
- Cost the split in data duplication, operational load, distributed workflows, and contract versioning. A case that cannot name those costs is unfinished.
- Map the seam to a driver. Ticketing versus nonticketing is a candidate because cadences and failure modes differ. Different nouns are not the argument.
- Prefer the smallest architecture that buys the driver. Package seams and bulkheads first, extra quanta next, a rewrite last. A rewrite is the case you make when the smaller seams cannot buy the driver.
- Time-box the first slice. "Extract reporting reads" or "stop sharing the tax table" can be funded. "Become a platform" cannot.

Put a fitness function in the case. "When this is done, billing tests run without dispatch, and CI forbids a new import of dispatch." A case with no check is declared done at the first new repository.

## When the case is ready

Split when at least one driver is already charging you, the seam is visible enough to attack in the code, and the payment — operations, data, latency — is smaller than that charge.

Wait when the pain is tests you could add inside the quantum, missing hardware, or a single hot query. Those are cheaper than a service cut. Wait when you cannot say which quantum would own the rows. Code split across teams that still share every table leaves one quantum, and it schedules more meetings. The releases next quarter still move together.

Argue ticketing against nonticketing, or against the two real workflows you have, using the five drivers. If none of the five is loud, there is no modularity case yet. Stay on one quantum, and spend the attention on fitness functions.

## What goes wrong

Each driver has a cut that misses it.

- Maintainability: a layer split, or a service so small that one change still crosses five repositories.
- Testability: a fleet whose integration tests take all night, or production used as the only signal. No new test can be named.
- Deployability: many pipelines still gated by one migration, or a shard whose real gap was a missing test. The dispatch fix still waits on the audit column.
- Scalability: a split with no load parameter, chatty services whose bottleneck moved to the network, or a capacity problem that vertical scale would have closed. Avatar-edit as a service, with no numbers, is this miss.
- Availability: three replicas of one combined process, a mesh that becomes the common failure, or two quanta re-joined by a synchronous call on the hot path. A bulkhead (its own pool, worker, or cgroup) can contain a failure without a new quantum. It is not enough when the data and the failure domain still have to part and cannot.

The business case fails as slides: principles, no numbers, no loser. It also fails as a precise cost model aimed at a cut that ignores static coupling, so the money buys a distributed monolith. Splitting "so teams can move independently," while the schema stays shared, skips independent deploy. The teams are not unlocked.

## Recap

Name one driver that is already charging you, and point at a workflow, a metric, or a release log. Maintainability wants a seam on change reasons. Testability wants a test that runs without the other workflow. Deployability wants a cadence the release log can show. Scalability wants different hardware for different load curves. Availability wants a blast radius that stops at the capability that failed. Cost the stay and the move, including duplication, operations, distributed workflows, and contract versioning. Take the smallest seam that buys the driver, and attach a check that fails if the seam is crossed again.

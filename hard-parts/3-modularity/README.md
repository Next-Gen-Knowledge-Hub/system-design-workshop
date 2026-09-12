# 3. Architectural modularity

Companion notes for **Chapter 3** of *Software Architecture: The Hard Parts*
(Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani; O'Reilly,
October 2021).

Chapter 2 taught you to *see* quanta. This chapter answers **why you
would pay to have more than one**. Skip it and decomposition becomes a
fashion: you will split a working monolith because "we need
microservices to scale," then discover you cannot test a workflow,
cannot ship billing without dispatch, and cannot explain the extra
latency to finance. Modularity is a response to named drivers — or it
is vandalism of a system that was allowed to be one quantum.

**See also (do not merge):**
[DDIA ch. 2](../../ddia/2-nonfunctional-requirements/) is how you
*measure* scale, latency, and reliability (percentiles, SLOs, load
parameters). This folder is *why you split a monolith* — the
modularity drivers that turn pain into a decomposition argument. Do
not paste an SLO in here and call it a business case, and do not paste
"maintainability" into DDIA and call it a percentile.

## The mental model

```
  one quantum (honest monolith / modular monolith)
       |
       |  is a driver actually hurting?
       |  maintain | test | deploy | scale | survive
       v
  pay for a seam (module, then maybe another quantum)
       |
       v
  you buy that driver, you pay coupling-of-a-new-kind
  (data split, workflow, ops, latency)
```

The one sentence to remember a year from now: **split only where a
modularity driver is real and costed**, not where the org chart or a
conference wants another box.

Two consequences fall straight out of that diagram. First, five drivers
are five *different* arguments — scaling assignment is not the same
claim as "we cannot test invoicing." Second, each split you win on one
driver you lose somewhere else (usually data and operations). The
business case is that comparison, not a slogan about agility.

## Drivers of modularity

A **driver** is a characteristic that is currently bad *because* too
much lives in one quantum, and that would get better if you put a seam
in a specific place. "Better architecture" is not a driver. "Netflix
does it" is not a driver. If you cannot point at a workflow, a metric,
or a release log, you do not have a driver yet.

Use the book's ticketing/field-ops case as the shared label when you
need a concrete pair of workflows; use your own product when you can.

### Maintainability

**Maintainability** is how safely a human can find and change the
behavior they meant to change. In a single quantum stuffed with two
cadences of work, a tax-table edit and a dispatch-rule edit compete
for the same attention, the same types, and the same review queue.

**Problem** — Every change is a treasure hunt. A new hire cannot tell
whether `Status` means ticket status, invoice status, or technician
shift status. Touching billing recompiles and re-risks assignment.

**Solution** — Put a seam where *change reasons* diverge. Maintainability
improves when the files, types, and tests for one reason live together
and do not mention the other reason. That seam can start as packages
inside one quantum ([ch. 5](../5-component-patterns/)); it does not
have to start as a service.

What you might measure without stealing DDIA's percentile toolkit:
files touched per change-reason, time-to-first-useful-PR for a new
teammate on *one* workflow, number of cross-workflow imports in a
diff. These are crude. They are still better than "the code feels
messy."

**Failure mode** — Split by technical layer (all controllers, all
repositories) and call it maintainability. You made navigation worse
and did not isolate change reasons. Or: extract a service so small
that a business change still hops five repos.

Production picture: a retailer whose "promotions" logic is copy-pasted
through checkout, cart, and email. Every campaign week is a multi-repo
hunt. The driver is maintainability (same change, many homes), not
scale. A promotions module — still one quantum if you need the
transaction — can be the honest first step.

### Testability

**Testability** is whether you can get a fast, believable signal that a
change is safe *without standing up the universe*. One quantum that
boots a database, a mailer, a search index, and last year's batch jobs
before it can unit-test tax rounding will be tested rarely and late.

**Problem** — The only honest test is an end-to-end run on a full
environment that is flaky and booked. People stop writing tests for
the part they changed. Regressions move to production.

**Solution** — A seam that lets you test a component or domain with
fakes at the boundary. Testability as a *modularity* driver is not
"raise coverage 10%." It is "invoicing can be proven without dispatch
running." Fitness functions from [ch. 1](../1-no-best-practices/) fit
here: a pipeline that fails if `billing` tests import `dispatch`, or
if the billing test target pulls in the whole monolith classpath.

**Failure mode** — Testing in production as a personality; or a fleet
of services whose *integration* story is harder than the monolith's
ever was. If you cannot name the test you can now run that you could
not run before, you did not buy testability. You bought YAML.

Production picture: a hospital-billing module that needed a full
patient-chart stack to assert a rounding rule. After a seam, the
rounding tests run in seconds on a laptop. The remaining e2e suite
shrinks to contract tests at the seam. That is the driver working.

### Deployability

**Deployability** is how often and how safely you can put a slice of
behavior into production. It is the operational twin of independent
deploy from [ch. 2](../2-coupling/): not "do we have two jars," but
"can invoicing miss this train without holding dispatch hostage?"

**Problem** — A release train. Ticketing bugfixes sit behind a finance
freeze. Or: every deploy is a full-system outage window because you
cannot tell what changed.

**Solution** — A seam that makes *cadence* independent where the
business already wanted independent cadence. Dispatch on Saturday,
tax tables on a weekday with extra review — that is a deployability
argument. Measure it with release logs: wait time per workflow, failed
deploys whose cause was "someone else's module," rollback scope.

A modular monolith can improve deployability only slightly (you still
ship one artifact) but can improve *confidence* (smaller diffs, module
test gates). If the pain is truly lockstep *releases*, you are arguing
for another quantum, not only another package.

**Failure mode** — Twenty pipelines that must still go green together
because of a shared migration. You automated the train; you did not
decouple it. Another: deployability used as an excuse to shard a
system whose actual pain was a missing test.

Production picture: a field-ops dispatch fix waiting three weeks on a
release because invoicing had an unfinished audit column. The driver
is deployability. The split that does not also split that column is
theater.

### Scalability

**Scalability** as a *modularity* driver means: **different parts of
the system want different amounts of hardware**, and they are stuck
sharing a process, a heap, or a connection pool. You scale the hot
path by also scaling the cold PDF renderer, or you scale reads by
also replicating a write-heavy ledger you did not want to.

This is not DDIA's question "what is p99 and which load parameter
grows?" You still *use* those measurements as evidence. The Hard Parts
question is: **does a seam let you scale one capability without buying
the same multiple for the others?**

**Problem** — Assignment spikes when storms hit; invoice PDF generation
is a lunchtime batch. They share a JVM. You add pods for the storm and
pay for idle PDF capacity all year — or the batch knocks over
assignment.

**Solution** — Cut where the *scale curves* differ, not where the
domain diagram is pretty. Sometimes that is a queue and a worker still
inside one company's network, not a public microservice. Sometimes it
is a read replica for reporting only. Put the cut on the curve, then
check static coupling so you did not leave both sides on one database
primary ([ch. 2](../2-coupling/), [ch. 6](../6-operational-data/)).

**Failure mode** — "We need to scale" without a load parameter. You
split into dozens of services, each too chatty to scale independently,
and the bottleneck moves to the network and the orchestrator. Or you
scale a monolith vertically and that was enough — there was no
modularity driver, only a capacity ticket.

Production picture: a chat product where search indexing CPU dwarfs
message ingest. Splitting the indexer (async, own hardware) is
scalability-as-modularity. Splitting "edit user avatar" into its own
service is not, unless you have numbers.

### Availability and fault tolerance

**Availability / fault tolerance** as a driver means: **a failure in
one capability should not take down another**, and today they share a
fate because they share a process, a memory space, or a blocking call.

**Problem** — A leak in reporting OOMs the monolith and dispatch goes
with it. Or a hanging call to a tax API freezes every HTTP worker,
including "create ticket."

**Solution** — Isolation: separate processes, separate connection
pools, timeouts, bulkheads, and — if you go as far as another quantum
— no sync call on the survivor's critical path. The driver is
*failure containment*, not uptime as a vibe. Evidence: incident
timelines where the blast radius crossed a workflow that did not need
to be in the blast.

Dynamic coupling from [ch. 2](../2-coupling/) is how you accidentally
give the isolation back. Two quanta plus a sync call on the hot path
is one failure domain with extra latency.

**Failure mode** — "High availability" meaning three replicas of the
*same* god process. You survived a node loss. You did not survive a
bad deploy or a poison query. Or: so many services that a mesh outage
*becomes* the shared fate you were fleeing.

Production picture: PDF generation of year-end statements pegs CPU;
intake dies. A worker pool with its own cgroup — still maybe one
quantum — is often the first availability seam. A new service is
justified when the failure domain and the *data* can actually part.

## Creating a business case for modularity

Drivers are engineering language. A **business case** is why the
company spends money and risk to act on a driver *now*, rather than
next year, and why *this* seam rather than a rewrite.

**Problem** — The proposal is "migrate to microservices" with a
capability map and no cost of staying, no cost of moving, and no
sequencing. Leadership hears fashion. Engineers hear a two-year freeze
on product work.

**Solution** — Write the case as a comparison of two futures, using
the drivers as line items. Keep it ugly and specific.

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

Rules that keep this honest:

- **Cost the status quo in incidents and wait time**, not in feelings.
  "We lost Saturday dispatch twice this year because of billing
  deploys" is a case. "Monoliths don't scale" is not.
- **Cost the split in the currency the rest of this book will charge
  you:** data duplication, operational load, distributed workflows,
  contract versioning. If you cannot name those, you have not read
  ahead far enough to sell the work.
- **Pick a seam that maps to a driver.** Ticketing vs nonticketing in
  the book's ticketing/field-ops case is a candidate *because cadences
  and failure modes differ*, not because they are different nouns.
- **Prefer the smallest architecture that buys the driver.** Package
  seams and bulkheads before extra quanta. Extra quanta before a
  rewrite. A rewrite is a business case of last resort.
- **Time-box the first slice.** "Extract reporting reads" or "stop
  sharing the tax table" is fundable. "Become a platform" is not.

**Failure mode** — Business case as slideware: principles, no numbers,
no loser. Or the inverse: a perfect cost model and a cut that ignores
static coupling, so the money buys a distributed monolith. Another:
splitting to "unlock the teams" when the teams would still share a
schema — you have not unlocked them; you have scheduled more meetings.

A fitness function belongs in the case, not in an appendix. "When this
is done, billing tests run without dispatch, and a CI check forbids
new `dispatch` imports." That is how [ch. 1](../1-no-best-practices/)
and this chapter lock together. The case without a check will be
declared done at the first new repo.

### Why you split — and why you wait

Split when at least one driver is already charging you, the seam is
visible enough to attack ([ch. 4](../4-decomposition/) asks whether
the code will let you), and the payment (ops, data, latency) is
smaller than the charge.

Wait when the pain is missing tests you could add *inside* the quantum,
or missing hardware, or a single hot query. Those are cheaper than a
service cut. Wait when you cannot say which quantum would own the
rows. Data ownership is [ch. 9](../9-data-ownership/); pretending you
can split code and share all the tables is how chapter 2's "one
quantum" diagnosis comes back.

The running case is useful here as a *discipline*: always argue
ticketing vs nonticketing (or your two real workflows) with the five
drivers, not with a generic "domain model." If none of the five is
loud, you do not have a modularity case yet. You have curiosity. Stay
one quantum and spend the curiosity on fitness functions.

## Check yourself

1. Pick a system you know. Which *one* modularity driver is actually
   charging you today? What evidence (release log, incident, test
   runtime) supports it, and what fake driver has someone proposed
   instead?
2. Why is "we need to scale" incomplete as a modularity argument until
   you name *which* capability and *which* load parameter? Steal the
   measuring stick from DDIA ch. 2 without merging the chapters.
3. Maintainability vs deployability: describe a change that is painful
   to *write* but easy to *ship*, and the opposite. Which seam helps
   which, in a product you have shipped?
4. A team adds twenty microservices and integration tests now take
   all night. Which driver did they claim, which did they worsen, and
   what does that do to the business case?
5. How can a bulkhead (separate pool, worker, cgroup) buy availability
   without buying a new quantum? When is that not enough? Give a
   production-shaped failure for each.
6. Write three lines of a business case for splitting ticketing from
   invoicing in the book's ticketing/field-ops case (label only): cost
   of staying, cost of moving, first seam. What fitness function
   proves the seam exists?
7. Someone wants to split so "teams can move independently," but the
   schema stays shared. Which chapter 2 property did they skip, and
   what will the next quarter's releases look like?
8. Testability as a driver: name a test you still cannot write after a
   bad split, and a test you *could* write after a good seam, using a
   real module you have touched.
9. Why might you refuse a split even when two drivers look real?
   Include data gravity in the answer.
10. Map each of the five drivers to a *wrong* cut (layer split, nano
    service, sync call on the hot path, etc.). Which failure mode is
    the one you have actually seen?

Continue to [Architectural decomposition](../4-decomposition/).

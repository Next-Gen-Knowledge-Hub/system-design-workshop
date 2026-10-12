# 15. Build your own trade-off analysis

These notes teach how to analyze a design seam, from *Software Architecture: The Hard Parts* (Ford, Richards, Sadalage, Dehghani, 2021).

## The method is the thing you owe

A question that looks like a tool choice is usually several entangled dimensions. The work is to name them, name the coupling points, and pick in public. Pattern names are inputs to that work. Reciting them is not the work.

A few names used below, defined here so the method has something to bite on:

- An **architecture quantum** is the set of parts that ship and fail together.
- A **saga** is an application protocol for a plot whose steps commit in more than one store: a state machine plus an undo story. **Fairy Tale** is the cell where a conductor tells each step to commit locally and runs compensations in reverse. **Time Travel** is the same visibility carried by events, with the undo as a second chain. **Anthology** is parallel eventual choreography: several stories, and "done" is a distributed opinion unless a completion rule exists. **Epic** is a conductor that hides half-writes until the end.
- **Stamp coupling** is depending on a composite payload or table the consumer did not want.
- **Data mesh** is domain-owned analytical products on a self-serve platform with federated governance. A **data product quantum** is the smallest analytical unit that is independently valuable and independently changeable.
- An **ADR** (architecture decision record) is the short written pick: what was chosen, the costs, what will be watched.
- A **fitness function** is a check that a constraint still holds, such as "ticket-open stays under 300 ms" or "billing can deploy without a ticket-service release."

Forks that belong to storage systems — transactional workload versus analytical scan, one machine versus a distributed store, and similar — can matter on a seam. Pasting that list into an ADR does not analyze the seam. There are no best practices to import. They return wearing conference badges. Catalogs of saga cells, reuse patterns, access patterns, and warehouse/lake/mesh are menus. A menu stops you inventing a ninth saga cell by accident. It does not pick the lunch.

"It depends" is the start. The output is: it depends on these dimensions; we picked this because that cost is the one we can pay. The method has five steps. A forty-cell matrix that ends in "it depends," a slogan, and a pitch with no trade-off are how the steps get skipped.

The running example is a real plot: open a ticket, charge the card, then assign a technician. A failed notice must not refund the charge. The review question that arrived was a boolean: "Should we choreograph?"

## 1. Untangle the dimensions

The boolean is a bundle. "Should this plot be choreographed?" hides at least five decisions, and they can land on different sides:

- Who owns workflow **state** — the row that says which step this run has reached.
- Whether half-writes may be **visible** — charged, not yet assigned — or must be hidden.
- How **fat** the event or command is.
- Whether charge and assign are one quantum or two.
- Which team gets the page.

Hybrid is what untangling is for: orchestrate the money path, choreograph the notice. If the matrix has one row, it has not started.

Find dimensions by walking the customer plot and marking four things. **Failure**: what happens when assign times out after the charge committed. **Change**: who ships when a fraud check is inserted. **Scale**: which box gets hot. **Ownership**: who may write the charge. A mark that is not forced by the others is its own dimension. If two marks always move together in this domain, they are one dimension here, even if a catalog listed them apart.

The failure mode is debating the boolean for an hour. The loudest dimension wins, usually latency or the word "decoupling," and the silent ones — the undo story, the stamp, the on-call — show up in production. The other failure is copying a book's dimensions as a checklist. Mesh versus warehouse does not need a saga cell. A saga cell does not need an argument about column stores.

## 2. List the coupling points

A trade-off table that never says who must change together is a product comparison. A **coupling point** is a seam where knowledge, deploy, or failure crosses a quantum.

For each option, and for each seam, ask:

- What knowledge crosses: fields, plot order, identity?
- What runtime crosses: a synchronous call, an event, a batch port?
- Who deploys when that knowledge changes?
- What fails when the other side is down or stale?
- Is the coupling **static** (you cannot start without them; the source names them) or **dynamic** (you couple when the plot runs)?

Compare options on the same points. On this ticket, three pictures:

Orchestration puts a conductor in front of billing and dispatch. The coupling point is honest and static: the conductor names the next worker, deploys when the order changes, and the plot is down when the conductor is down. Billing does not know dispatch.

Choreography removes that source-level name and raises a semantic hub: every subscriber has to understand "after charge, assign." The hub is harder to see, which is why it gets described as decoupled. A single stamped event that carries the whole ticket can couple more teams than five tiny commands. The coupling moved from "who do I call?" to "whose document did I just absorb?"

A thin event plus a read moves it again, from payload shape to extra reads. The owner must be up, and the data is fresh. Mesh versus a central warehouse is the same kind of move on analytical data: coupling leaves a central model and lands on federated contracts. A contract that is `SELECT *` into a lake is still a coupling point, and it is not a network call. There is no uncoupled design. There is coupled where you can stand it.

Write the constraint as a fitness function or as a clause in the ADR. "Ticket-open stays under 300 ms." "Billing can deploy without the ticket service." Analysis that cannot become a test or a sentence you will re-read is a conversation you will re-have.

The failure mode is counting. Topics, services, and repositories are easy to count and mostly the wrong metric. Who must change together is the metric.

## 3. Run the cases you actually have

"Service A calls Service B" makes every option look fine. Real plots have an irreversible step, a human, and a support org. Analyze on your cases: ticket-open, refund, technician no-show, "the customer merged two accounts." Walk the happy path, the timeout, and the compensation. If an option dies on a case you run, it is dead, however clean it looked on a generic slide.

On this ticket the interesting case is 3 a.m., not checkout's happy path. Assign times out after the charge has committed. A timeout does not mean the charge failed to land; partial failure — one side committed, the other did not, or one side is simply late — is where the options stop looking alike. Fairy Tale has one row that moves to `compensating` and refunds, or retries assign if finishing is cheaper (forward recovery). Time Travel needs a second event chain someone remembered to subscribe to. A design copied from a talk whose plot was optional notifications dies here, because this plot moves money.

The book's ticketing firm is a method: keep the plot, change the architecture under it, see what breaks. One toy case, especially the happy path, is the failure mode.

## 4. Say what you gain and what you pay

For each option that survived, write the gain and the cost on the dimensions you kept. Drop dimensions that do not move. If both remaining options are eventual, stop talking about atomicity.

Rank costs by who pays: the customer, the on-call, the team that cannot ship, the team that cannot understand the rewind.

A trade-off is assessed when a reasonable colleague can disagree with the pick and still agree on the costs. If you cannot agree what the costs *are*, the bundle is still tangled.

**Quantitative** when a number would change the pick: the p99 of the plot, the cost of a stamp at the fan-out you planned, the abort rate, a freshness SLO. **Qualitative** when the unit is judgment: cognitive load, irreversibility, regulatory shame. Write the sentence. Do not score "team sanity" as 3.7. Do not leave "latency" as a feeling when you have traces. Swapping the two launders a gut pick through fake precision, or ignores a measurement that would have moved the decision.

The failure mode on the numbers is a weighted scoring model. Invented weights make a gut pick look like arithmetic. If you already know the pick, write the ADR. If you do not, a fake formula will not discover it. The failure mode on the pile is a 20-by-20 matrix of traffic lights that ends yellow everywhere. That is fear in spreadsheet form. Assessing only the happy path is the same fear: the cells diverge on the undo.

## 5. Lead with the bottom line

Stakeholders need the choice, the one or two decisive costs, and what you will watch. The matrix goes in an appendix. An ADR that cannot be quoted in a standup is too long. A pile of evidence is how a pitch looks thorough, and how a scared review looks rigorous.

A bottom line for this ticket, once the dimensions above are accepted, can be quoted in one breath: we will orchestrate charge-then-assign in one conductor, with local commits and a refund compensation keyed by saga, step, and direction; the notice is a separate reaction that cannot undo the charge; we will watch assign's timeout rate and double-refunds. Someone can prefer a merge instead, and still agree those are the costs of the split.

The failure mode is thirty pages and no sentence that starts with "We will." A conclusion that restates the options is the same failure. The pick was hidden.

## Hygiene on the option list

**MECE** means mutually exclusive and collectively exhaustive. Options should not be the same idea in two costumes. The set should include do nothing, merge, and "not our problem" (change the product rule) when those are real. Dimensions should not be double-counted: if "coupling" and "independent deploy" are the same knob on this decision, list them once.

A MECE cut for charge-then-assign, on the axis "who owns the commit and the next step":

1. Change the product rule so you do not charge until a technician is assigned. The distributed plot goes away. That is "not our problem."
2. Merge billing and dispatch so both writes commit in one database. If they were never split, leaving that one service in place is the do-nothing case of the same commit boundary.
3. Keep two services and run a real two-phase commit, with prepares and a coordinator (the atomic orchestrated cell, Epic, or a parallel prepare if the steps are independent).
4. Keep two services, one conductor, local commits, compensations in reverse (Fairy Tale).
5. Keep two services, no conductor, each step commits and emits, and the undo is its own event chain (Time Travel).

A colleague who immediately names a missing option means the list was not exhaustive. "Use Kafka," "use Rabbit," "use HTTP," and "use gRPC" are not extra options on this axis. They are pipes. You can carry (4) or (5) on any of them and still have no design. MECE is hygiene, not a proof.

## The out-of-context trap, and pitches that hide the cost

"Netflix does it," "the paper says," and "best practice" are the same move. Steal dimensions and failure modes. Leave the conclusion where you found it, unless you know the source's quantum, org shape, scale, and compliance regime. If you do not know, you cannot import the pick. Their choreography assumed a platform team you do not have. Their warehouse assumed a finance close you do have. "Always choreograph" is not even evaluable until those are known. Copying a saga cell from a talk about optional notifications onto a plot that moves money is the trap. Copying mesh from fifty domains into a company with two is the trap.

Vendors and movements sell the absence of trade-offs. Microservices will decouple you. Events will decouple you. Mesh will scale you. Two-phase commit will make the plot ACID again. A new encoding will make contracts safe. Any "always" or "never" about an architecture style is a candidate for the bin. Ask what the pitch hides. It is usually a platform team, a visible inconsistency, a stamp, or a central bottleneck that moved and did not disappear.

Using a catalog as named costs is the opposite of that pitch. Using the catalog as a conclusion is the pitch with a nicer name. "We picked Anthology" without a join rule is still a pitch: Anthology's hidden cost is that nobody owns "are we done?" Fairy Tale, used as a conclusion, hides the hub, the visible half-state, and a compensation that can itself fail. Data mesh, used as a conclusion, hides the platform, the product work inside domains, and automated policy; without them you have twelve snowflakes or twelve swamps. A strict contract, used as a conclusion, hides the coordination cost, and a strict schema that copies the table still leaks the owner's innards.

Replacing last year's slogan with this vocabulary, without the five steps, is the same failure.

## Recap

Untangle the boolean into dimensions, list who must change together at each seam, run the cases you actually have (including timeout and undo), keep numbers and judgments in their own units, and write a bottom line a standup can quote: the choice, the decisive costs, what you will watch. Pattern names — Epic, Fairy Tale, stamp coupling, a data product quantum — are examples of named costs. The next decision may have no cute name: a new compliance rule, a vendor you cannot leave, a team that does not exist yet. The same five steps still apply. Reciting the cube finishes the glossary. The artifact a design review is owed is that method, recorded as an ADR and, where a constraint is real, as a fitness function. A catalog, a topic count, and a slogan are not the artifact.

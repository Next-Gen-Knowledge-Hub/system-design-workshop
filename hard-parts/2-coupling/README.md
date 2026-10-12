# Discerning coupling

Software Architecture: The Hard Parts, Ford, Richards, Sadalage, Dehghani, O'Reilly 2021, Chapter 2.

## What a quantum is

An architecture trade-off is usually about **coupling**: which things must change, ship, or fail together. An **architecture quantum** (plural **quanta**) is the smallest thing you can deploy independently that still hangs together. It is cohesive in purpose. **Static coupling** glues it on the inside. It talks to other quanta only through **dynamic coupling** you can name.

"Service," "module," and "bounded context" get used for wishful diagrams. A quantum is the unit you can check. It has four properties. Miss one, and the description is a hope.

"Loosely coupled" needs a kind. Static coupling is source, schema, a contract, or a shared library: anything visible without running the system. Dynamic coupling is a runtime call, a message, or a temporal need for the other side to be up.

A process that cannot ship without another team's database migration is a second process inside one quantum. A count of Git repositories, Kubernetes deployments, or bounded contexts on a slide does not change the count. A schema change, a shared-library bump, or a synchronous call on the checkout path is the evidence that one thing still had to move together.

The caller's experience of partial failure is what dynamic coupling feels like. The quantum is defined by what must still deploy together.

The field-ops product used below has ticketing work (intake, assignment, dispatch) and nonticketing work (invoicing, parts). Both mention customers. A shared noun is not yet a quantum boundary.

## Independently deployable

You can release this artifact through production without releasing the others as a coordinated set. Shipping on the same day by coincidence is allowed. Required lockstep is a single release.

Independence is an operational fact. Take the last six production releases and list the artifacts that had to move in one change request. That set is the quantum, even when the org chart shows three teams.

Rollback belongs to the same property. If A cannot be rolled back without rolling B back, the two failure stories are still synchronized.

On the field-ops train, billing, dispatch, and a shared platform jar go out together because a column default or a DTO field ships in all three. That set is the quantum.

Splitting the build and keeping a single migration folder multiplies deployables. Deploy independence stays where it was. Friday still needs a war room.

## High functional cohesion

The code and data inside the quantum do **one** business capability a domain expert would name in one breath: assign a technician, issue an invoice, reserve a part. Cohesion means this code and data change for the same reason. Classes that share a utility package are a packaging choice, not a capability.

Group by change reason and by workflow. Ticketing assignment and invoicing are different reasons to ship, even when both rows carry `customer_id`.

A "customer service" that also renders PDFs, prices tax, and pages on-call, because those screens sit in one app, mixes those reasons. **Accidental cohesion** groups everything that touches HTTP, or everything that touches `Customer`. You cannot scale, test, or replace a capability, because the clump was a layer.

Cohesion is what makes independent deploy worth paying for. A 40-line true/false service can be independently deployable and still useless. A technical layer is not a quantum.

## High static coupling

**Static coupling** is the coupling you can see with the system stopped: source imports, compile-time references, a shared database schema, a shared library version, a contract that must change in lockstep, a foreign key you cannot migrate alone.

Inside a quantum, static coupling should be high. Classes, tables, and internal APIs are supposed to know each other. That is cohesion with a consequence. High static coupling *across* things you are calling separate quanta is the defect.

Wanted, inside one quantum:

```
  assign.py --> slot.py --> work_orders table
  one schema, one ship, one rollback
```

Unwanted, across a hoped-for cut:

```
  assign.py --> invoices table
  assign.jar --> common-1.8.jar <-- billing.jar
  two service names, one quantum
```

Putting an HTTP boundary in front of the same tables and the same shared kernel leaves the static coupling in place. Inventory it the way you inventory secrets. For each pair of hoped-for services, list:

- shared database objects
- shared libraries that carry domain types
- generated stubs that break when either side moves
- foreign keys
- common packages that both sides compile against

A non-empty list means these are not two quanta yet. They are a distributed monolith, or one quantum with a network hop.

Data gravity is this list made concrete: the shared table, the shared key, the migration both sides must apply.

A count of REST calls measures a different thing. Few calls and one schema is still one quantum. Many asynchronous messages and separate data stores may be several quanta.

A build can watch the inventory. It fails when a new static edge appears between two claimed quanta: a helper import, a new shared type, a new foreign key. That is the leak the next small helper change would otherwise become.

## Dynamic coupling between quanta

**Dynamic coupling** is what happens at runtime between quanta: a synchronous call, an asynchronous message, a workflow that waits, a cache fill, or a rule that this side cannot start unless the other side is up. A useful system with more than one quantum keeps some of it. The shape is the operational contract, so name the shape.

- **Synchronous versus asynchronous.** Synchronous: the caller waits until the callee answers or the call times out. The callee's availability is now inside the caller's SLO. Asynchronous: the caller has taken on lag, and the case where the message never arrives.
- **Temporal.** An asynchronous link is still temporal when the workflow cannot proceed until the other side processes the message. A queue moves the failure from HTTP 503 to a stuck ticket. The coupling remains.
- **Data versus control.** Passing a document the sender already owns is different from asking another quantum to decide. Control-style coupling — orchestration callbacks, or "is this technician allowed?" — makes two quanta one business transaction in disguise.

Events can carry both kinds at once. The consumer cannot deploy until the producer's new required field ships: that is static coupling riding with the message. The UI still waits on the consumer's handler: that is temporal coupling. Draw each runtime arrow and label sync or async, who waits, who owns the data in the payload, and what an hour of absence does. That drawing is the dynamic quantum coupling.

Dynamic coupling between quanta is allowed. It is how more than one quantum cooperates. Static coupling must not hide inside the call.

Rewriting REST as Kafka and keeping the shared `customers` table changes the dynamic edge and leaves the static one. The system gained an operations surface. The quantum count stayed the same.

## Count quanta from data and releases

The number of quanta is often smaller than the number of boxes. The same counting works on a monolith, a modular monolith, or a fleet of services. Count processes last.

1. List the deployables that are supposed to be independent.
2. Cluster them by static coupling: shared schema, shared domain library, lockstep contract, migrations that must ship together. Each cluster is a candidate quantum.
3. Check cohesion. A cluster with two change-reasons — tax tables and on-call assignment — is under-split *inside* the quantum. That is a decomposition problem. It is not evidence that two quanta already exist.
4. Label the arrows that remain as dynamic coupling. Those arrows are the architecture between quanta.

```
  hoped-for:   [intake] [assign] [invoice] [parts]
               four "microservices"

  static:      intake + assign + invoice + parts
               on one OLTP database
               ---------------------------------
               one quantum, until the database splits
```

A modular monolith can be one honest quantum: independently deployable as a unit, cohesive enough inside, highly statically coupled, with no other quantum to talk to. That is a valid architecture. It becomes a false description when the modules are claimed to deploy independently and they cannot.

A service-per-class fleet is one quantum when the services share a kernel and a database. Extra YAML does not mint a quantum.

Independent deploy means a different release cadence is possible, rollback is possible, and a fatal GC pause in invoicing does not have to kill assignment. It does not mean any of the following.

- A separate process. Many processes can be one quantum.
- A separate team. Team topology should follow quanta. Two teams on one quantum pay a coordination tax. One team on five accidental quanta pays a paging tax.
- An API. Any class can have an API. The test is whether the other side survives a Tuesday on which you do not ship.
- An asynchronous hop. That is one shape of dynamic coupling.

A synchronous call placed back on the hot path returns the shared fate. Dynamic coupling spends the availability the split was supposed to buy.

## Assignment and invoicing

Apply the four properties to the field-ops cut.

Shared `customers` and `work_orders` in one schema put high static coupling across the hoped-for boundary. One quantum.

Extract invoicing to its own deployable, and have assignment call it synchronously on every close-ticket. The result is a process boundary, and assignment's availability now includes invoicing. Two candidate quanta, with tight dynamic coupling. The cut may be wrong. It may be a step you mean to leave. Name which one it is.

Have invoicing consume `TicketClosed`, own `invoices`, and ship on a different day from assignment. That approaches two quanta. The price is lag and a reconciliation story. The price is the trade-off.

A design review states the number of quanta and the evidence: release sets, shared schema, shared domain libraries, lockstep contracts. A count of repositories is a different number.

Maintainability, testability, deployability, scalability, and fault isolation are reasons you might want more quanta. Counting boxes first, and splitting toward those reasons before the four properties hold, produces more processes around the same quantum.

## What goes wrong

The release train looks like three teams and is one change request. The build is split, the migration folder is shared, and Friday is a war room. Rollback of one artifact forces rollback of the other.

Cohesion by layer, or by every type that mentions `Customer`, yields a clump you cannot scale, test, or replace as a capability. The other edge is an independent service so small that no domain expert would name it.

An HTTP facade on one kernel and one schema is a distributed monolith. A REST-call count hides it. A bus in front of a shared `customers` table does the same: static coupling stays, and the bus is new operational work.

A queue the workflow must wait on still couples the sides in time. The visible failure moves from HTTP 503 to a stuck ticket. A control call such as "is this technician allowed?" hides one business transaction behind two processes.

Describing a modular monolith as independently deployable modules, when those modules cannot ship apart, is a false quantum count. A class-sized fleet on one database is one quantum with more deploy manifests.

## Recap

Check four properties: independent deploy including rollback, one named capability, high static coupling inside, and only named dynamic coupling outside. List deployables, cluster by shared schema, shared domain libraries, lockstep contracts, and joint migrations, then reject a cluster that still holds two change-reasons. Label every remaining arrow with sync or async, who waits, who owns the payload, and what an hour of absence does. On this product, shared `customers` and `work_orders` keep assignment and invoicing in one quantum until that sharing ends.

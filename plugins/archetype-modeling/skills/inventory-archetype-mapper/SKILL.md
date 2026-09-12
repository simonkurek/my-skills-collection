---
name: inventory-archetype-mapper
description: Transform domain requirements into an Inventory (resource availability) Archetype model. Identifies the complexity level (0–8), resources and their availability kind (exclusive, divisible, time-partitioned, bundled), claims and ownership, hold expiry, reservations, shortage policies and wait lists, stock identity and consumption, and the boundaries with product, ordering, party and accounting. Produces an implementable model with explicit concept mapping, unmapped concepts and integration boundaries.
argument-hint: "[domain requirements or feature description]"
---

# Inventory Archetype Mapper

Transform any domain description in which requests compete for something limited — goods, specimens, time slots, capacity, people, licences, limits — into an Inventory archetype model. Inventory answers one question: *"can we commit this resource, with these parameters, right now — and who holds it?"* It knows resources, their availability, claims, holds that expire, pools, reservations and wait lists. It does not know why something was reserved, what it costs, or who is allowed to ask.

**Output goal**: A complete, implementable model that says what the resources are, how their availability is computed, who may claim and release them, how holds expire, how reservations group claims, what happens when there is not enough, and where Inventory ends and Product, Ordering, Party and Accounting begin.

## When to Use

**Use this skill when:**
- Selling, performing or lending something depletes a countable supply (units leave the warehouse, free slots leave the calendar, litres leave the tank)
- Two requests can compete for the same thing, and the same request can succeed at one moment and fail at the next without the customer, product or price changing
- Something is held for a while and then released, expires or is consumed
- Someone must wait until something frees up, and waiting can resolve itself
- The business asks *"do we have it?"* separately from *"may they buy it?"* and *"what does it cost?"*

**Output is useful for:**
- Domain modeling sessions before implementation
- Deciding which module owns availability when Ordering, Product and Inventory are being separated

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the inventory archetype does not fit, and briefly explain why.

### The core question

> *"Can I ask 'can we commit resource R — a specimen, an amount, or a time window — to S right now, and who holds it?' and does the answer change as others claim and release it?"*

If **yes** → inventory archetype likely fits.

Counter-questions that route elsewhere:
- *"How much X does S have, with a transaction history?"* → **accounting**. A completed movement with history and audit is a ledger; a claim on the future is inventory. The two often coexist: inventory holds the promise, accounting books the movement.
- *"What kind of thing is it, and what values are allowed?"* → **product**. Product says what and what is permitted; inventory says whether, how much, which one, when.
- *"What can this party do, where, when, up to what volume?"* → **party**. A declared capability is potential, not a reservation system. *"Whether Wednesday 10:00 happens to be free will be decided by another module."*
- *"Who asked for what, on what terms, and is it confirmed?"* → **ordering**. The order describes intent; the reservation describes availability of resources over time. The order may request a reservation but never owns it.
- *"What state is X in?"* → a state machine, not availability.
- *"What does it cost?"* → **pricing**.

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "no capacity today — the customer waits until something frees up" | ✅ Yes |
| "one slot, one owner; you cannot book a taken slot" | ✅ Yes |
| "the seat is held until payment, then returns to the pool" | ✅ Yes |
| "several companies draw from the same tank as long as the total fits" | ✅ Yes |
| "reserve the room for three nights — all or none" | ✅ Yes |
| "we need to know which serial number / which batch went out" | ✅ Yes — stock identity layer |
| "the product does not exist before the order (account, policy, subscription)" | ❌ No — the order creates it; nothing is depleted |
| "offered to VIP customers only / not sold in this country" | ❌ No — applicability rule, belongs to product |
| "Maria can do ultrasound, Mon–Fri, up to 15 patients a day" | ❌ No — party capability |
| "the reservation lives in a partner's system" | ❌ No — external allocation; inventory does not take part |
| "data allowance: granted, used, blocked, expires; history matters" | ⚠️ Borderline — ask: is the history of grants and consumptions the point? then accounting |
| "requests conflict because they share power, cooling, a test rig" | ⚠️ Borderline — the calendar is inventory; the indirect conflict is a graph problem (see Step 10) |

### Borderline cases — how to decide

- **Slots**: an appointment slot is inventory (a time-partitioned resource with one owner), the visit that results is a product instance, and the order that requested it is a state machine. Three things, three homes. Do not pre-create a product instance for every possible slot.
- **Queues**: a wait list for a scarce resource is part of inventory's shortage handling. A queue of *change requests* over slots ("swap me with whoever holds Tuesday") is a graph problem (cycles = realisable swap groups); do not grow the reservation model to solve it.
- **Potato depot / data allowance**: if the business wants balances with a full history of intake, reservation and issue, model the movement in accounting. The sources model a whole reserve-hold-expire-collect cycle over a physical quantity as a ledger, so an expiring hold on its own does not force inventory. What forces inventory is contention — two requests racing for the same unit, slot or amount — and then the promise lives in inventory while the movement may still be booked in accounting.
- **Pools of numbers (bank accounts, phone numbers)**: inventory when the question is "is there one free, and may we expand?"; identity, not inventory, when the question is "whose number was this a year ago?".

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The inventory archetype requires a limited resource that requests compete for, that can be
claimed and released, and whose availability changes over time. This domain is a
[state machine / applicability rule / party capability / ledger / ...] because:

- [specific reason from the requirements]
- The natural question is "[what state / how much with history / what is allowed]", not
  "can we commit R to S right now, and who holds it?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — what limited thing do requests compete for, how is it held and released, and what happens when there is not enough?"

---

### Step 1: Assess Complexity Level

Identify **every layer that has a signal** in the requirements. Level 1 is the foundation; levels 2–8 are largely independent above it — a domain may need timed holds without time slots, or wait lists without stock identity. Model each layer with a signal, **and no further**. Report the level as the highest one modeled, but adopt layers by signal, not by number.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 0 | **No scarcity** | Nothing is depleted (e-book, generic service); "unavailable" means a rule, not a shortage | (none — a product rule or a status field is adequate) |
| 1 | **Claimable resource** | Something is taken and given back by an identifiable holder; either one holder at a time (exclusive) or many holders sharing an amount (divisible) | an `isBooked` flag or a counter decremented in place; nobody knows who holds what, so nobody can release it |
| 2 | **Time-partitioned availability** | The same resource is claimed per period — nights, hours, appointment slots; a range must be free as a whole | a booked flag per calendar row; range bookings that fail halfway and leave nights taken |
| 3 | **Holds that expire** | "held until payment", "reservation lapses after 30 minutes", "late confirmation is rejected" | a cron job deleting rows; stale flags; the caller keeps the timeout logic |
| 4 | **Composite demand** | Several resources or several periods must be taken together — room + parking, three nights, car + child seat | sequential locks that leave a partial bundle standing on failure |
| 5 | **Reservations as business objects** | Claims are grouped under an owner with a purpose and a status; cancellation is compensation; partial release; queries "what does owner X hold?" | claims scattered across tables; the order owns availability logic |
| 6 | **Shortage handling** | Unavailability is not failure: waiting, retry, growing capacity; who is served first matters | a "pending" flag; first-come-first-served that ignores priority or fit |
| 7 | **Stock identity and consumption** | Which specimen (serial), which lot (batch), how many (units); held vs offerable; reserved vs taken out; deliveries | one quantity column for everything; serials in free text; stock count silently used as capacity |
| 8 | **Contention beyond the calendar** | A free slot is not enough: shared power, cooling, acoustics, a shared balance; "how many parallel lanes do we need?" | a bigger reservation model (route to a graph model instead) |

**Guidance — which steps to run:**
- Level 0: the archetype is overkill. Emit the ❌ Does Not Fit block from "When NOT to Use" and stop; do not produce a model. Say: "nothing is depleted and nothing is held — a product rule or a status field is adequate; the archetype would add ceremony without buying anything."
- Always: Steps 0–3, Step 4 (resources and availability), Step 5 (claims and ownership), the demand-shape part of Step 7, the allocation-timing part of Step 8, the boundaries part of Step 10, and Steps 10.4–10.5.
- Then run one step per **signalled** layer, regardless of the numbers in between: time-partitioned or expiring holds → Step 6; composite demand → the groups part of Step 5; reservations → Step 7; shortage handling → Step 8; stock identity or consumption → Step 9; contention beyond the calendar → the graph part of Step 10.

Partial adoption is normal. Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps between the requirements and the inventory archetype. Ask about **two categories** in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more are needed).

#### Category A — Standard Inventory decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as **design choices**:

- **Exclusive or divisible**: Is the resource held by one party at a time, or do several parties share it, each holding an amount, as long as the total fits?
- **Whole or per period**: Is the resource claimed as a whole, or per time slot (night, hour, appointment)? Who defines the slots?
- **Hold lifetime**: Does a hold last until released, or expire after a deadline? What happens at expiry — the unit returns to the pool, someone is notified, the waiting list is checked?
- **Who may release**: Only the holder? Also an operator override? Can a hold be transferred to another party?
- **Shortage policy**: When there is not enough — reject, hold for a limited time, queue and wait, retry later, or grow capacity?
- **All-or-nothing**: When a request spans several resources or periods, is partial acquisition acceptable, or must it be all or none?
- **Allocation timing**: Is the resource claimed before the request exists (slot chosen first), at confirmation, in an external system, or never? Does this differ per product, channel or process?
- **Reserved vs consumed**: Does the domain distinguish holding from taking (pick-up, fuelling, check-in)? Does consumption reduce what can be promised later?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the inventory archetype supports but the requirements do not mention**. For each gap found, ask whether that dimension is wanted. Do not limit yourself to this list — reason freely:

- **Identity of what is claimed**: Does anyone need to know *which* specimen went out, or only *how many*? Are there lots that matter for recall, expiry or quality?
- **Re-claiming one's own hold**: If a holder asks again for what they already hold (extend, re-confirm) — accept, merge, or refuse?
- **Partial release**: Can a holder give back part of an amount, or one period of a multi-period stay?
- **Choice of resource**: When a request names a type ("a diesel tank", "any room") rather than a specific resource, who chooses and by what rule — first free, nearest, oldest lot first, load balancing?
- **Wait list order**: Arrival order, priority, first request that fits, or a quota per customer segment? Who triggers the re-attempt when capacity frees — inventory itself, or the caller on a schedule?
- **Capacity origin**: Is capacity declared (a tank holds 5,000 L) or derived from stock on hand (we have 37 units)?
- **Units**: One unit per resource? Is conversion between units required (kg wholesale, pieces retail)?
- **Blackouts**: Must a resource be unavailable for reasons other than a claim — maintenance, closure, not yet set up?
- **Over-booking**: Is deliberate overselling wanted (no-show buffers)? *(Not covered by the source material — if wanted, model it as an explicit allowance on the pool.)*
- **Purpose**: Does the business distinguish a customer booking from an internal allocation or a technical hold, for reporting?

Collect answers before proceeding. If the user cannot answer, document the assumption made in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

Always include **"To zależy / It depends"** as an explicit option in every `AskUserQuestion` call — do not rely on the automatic "Other" fallback. Place it as the last option in each question. If the user selects it, treat it as a **policy supplied from outside inventory**:

- Name the *parameter* inventory receives with the request (hold duration, shortage policy, all-or-nothing flag, selection rule, allocation timing)
- Note in **Implementation Notes** which module decides it — most often the product definition, sometimes the channel or a rules module — and that inventory only applies it
- Do **not** model the decision logic inside inventory

This is the intended outcome: *"the policy is passed explicitly; inventory does not guess whether to reject, queue or retry."*

#### If `AskUserQuestion` is not available

Do not stop. For every question you would have asked, pick the option most consistent with the requirements, mark it **(X — confirm)** in the Clarifying Questions & Answers section, and continue. Defaults if the requirements are silent: exclusive unless amounts are mentioned; whole resource unless periods are mentioned; holds last until released; only the holder releases; shortage = reject; all-or-nothing; allocation at confirmation; holding and taking not distinguished; re-claim by the same holder accepted; no partial release; no over-booking. Step 10.5 applies the same policy — it does not add a second one.

---

### Step 3: Map Domain Concepts to Inventory Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Inventory Archetype | Notes                          |
|----------------------|---------------------|--------------------------------|
| [domain noun/verb]   | Resource / Availability (kind) / Claim / Owner / Hold duration / Time slot / Reservation / Demand shape / Shortage policy / Wait list / Selection policy / Item type / Instance / Batch / Consumption | [why] |
```

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear inventory archetype equivalent:
- [concept] — [reason it doesn't fit / decision needed / which sibling archetype owns it]
```

This section must be present even if empty (`None identified`).

---

### Step 4: Define Resources and Their Availability

**Resource.** The thing whose availability is tracked. Inventory describes it generically — *"it does not know what a doctor, a courier or a bank account is"* — as a resource type, a resource identity, and availability parameters: time, place, duration, quantity. The UI and the business process translate domain language into resource language at the edge. If a resource cannot be expressed this way, ask what is missing and whether that missing thing is really inventory's.

**Availability.** For each resource, one or more availability records answer: *is it free now? how much? who holds it?* Availability is a **function of the claims on the record, their durations and the current time** — it is computed, never stored as a flag.

**Rule:** a resource may have many availability records (one per time slot); a record belongs to exactly one resource.

| Kind | Meaning | Capacity | Claims held | Typical resources |
|------|---------|----------|-------------|-------------------|
| **Exclusive** | one indivisible thing; winner takes all | 1 / 0 | at most one live claim | a laptop, an MRI scanner, a parking space, a unique artwork |
| **Divisible (pool)** | interchangeable units; many holders share it | free = on hand − Σ active claim amounts | many, each with an amount | a fuel tank, API tokens, consulting hours, account-number pools |
| **Time-partitioned** | one thing during one period; exclusive within the slot | 1 / 0 per slot | at most one per slot | a room per night, a doctor per appointment, a court per hour |
| **Bundle** | several resources that must be taken together | free only if every component is free | one claim per component, or one composite claim mapping to them (Step 5) | car + child seat, room + breakfast + parking |

Exclusive and time-partitioned are the same mechanism; time only multiplies the records. The real axis is **exclusive vs divisible**; a bundle is a rule over other records, not a fifth storage kind.

**Divisible capacity.** **On hand** = the amount physically held now: declared capacity or the summed quantity of mapped stock (Step 9), less everything already consumed, plus everything delivered. **free = on hand − Σ amounts of active claims.** State whether on hand is declared or derived (Step 9), whether consumption exists in this domain (Step 9), and that an expired claim stops counting immediately (Step 6). A request for exactly the remaining amount succeeds; a request larger than the remainder fails, even if smaller than total capacity. Over-booking is outside the source material; if wanted, add an explicit allowance to the formula and say so.

**Blackouts.** A period with no availability record is simply not claimable — closed and never-set-up look the same. If the business must tell them apart, or needs maintenance holds, model a blackout as a claim by a system owner or as an explicit record status, and say which.

**Records into:** the Resources & Availability section.

---

### Step 5: Define Claims and Ownership

**Claim** (lock, hold, block). A recorded hold on one availability record: identity, **owner**, the instant it was taken, its **duration** (Step 6), and for a pool its **amount**. A claim does not know why it exists — purpose lives on the reservation (Step 7), and *"inventory need not know why something was reserved."*

**Owner.** An opaque party identity — a guest, a fleet company, a department, an order. It is the sole authorization concept inside inventory.

**Rules:**
- A claim always has an owner, a start instant and a duration.
- **Only the owner may release a claim.** Anyone else is refused and the claim stands.
- An exclusive record or slot admits at most one live claim; another party's live claim always conflicts.
- A release makes the capacity available to everyone else — after the release, never speculatively before.
- Different resources are independent: claiming one never affects another, even for identical time windows.

**Convention:** re-claiming what one already holds (extending, re-confirming) is accepted, not treated as a conflict. Alternative: refuse and require release-then-claim. Decide whether the extension keeps the original claim identity or issues a new one — a reservation holding the old identity must still be able to release it.

**Convention:** releasing an unknown or already-released claim fails rather than succeeding silently, so a caller that has lost track of a claim learns it. Alternative: idempotent release, kinder to retries. Not settled by the sources — decide and say which.

**Operator override.** Not in the source material. If the business needs force-release or transfer of a hold to another party, add it as a distinct operation with its own authorization and say that it lives above inventory's owner rule.

**Groups and bundles (run if composite demand is signalled).** A request that spans several records — three nights, room + parking — is **all-or-nothing**:
1. check that every component is free for this owner;
2. take them one by one;
3. on the first failure release everything already taken and fail the whole request with the accumulated reasons.

Partial acquisition is never surfaced. The result is one claim per component (the group can be released as a unit) or one composite claim mapping to the component claims. State which, and state whether the group-level release must succeed for every component or may leave a partial state.

**Records into:** the Claims & Ownership section.

---

### Step 6: Define the Time Model

*(Run if time-partitioned availability or expiring holds are signalled — levels 2–3.)*

Two independent time axes must not be confused:
- the **time slot** of a record — *which period of service is held* (the night of 15 June);
- the **hold duration** of a claim — *how long the promise lasts* (20 minutes while the guest pays).

A time-partitioned claim has both: "hold the night of 15 June for 20 minutes."

**Time slots.** A half-open interval `[from, to)`. Two slots conflict when they overlap; adjacent slots (one ends where the next starts) do not. **Rule:** one slot = one availability record = one claim; multi-period demand is a group of claims (Step 5), never one claim with a range.

**Slot origin.** Slots are materialised — a resource's calendar *is* the set of records registered for it. State who generates them (opening hours, a schedule, a manual roster) and at what granularity; a request must name a slot the calendar knows. Partial-slot booking, splitting and merging are outside the source material; the simple move is to regenerate the calendar — if the business needs them, model them and say you went beyond the sources. *"For temporal resources we do not create an instance for every possible slot"* — the product instance (the visit) comes into being after the reservation, not before.

**Hold duration.** Either **permanent** (until the owner releases) or **timed** (expires at start + duration). The owner's confirmation may convert a timed hold into a longer or permanent one — an offer becoming a booking. A timed claim stops consuming capacity the moment it expires, before any cleanup runs. **Decision:** how expiry is acted on —
- *lazy*: the next claim simply overwrites the expired one;
- *swept*: a periodic job releases expired claims and reports their identities;
- *event*: expiry publishes an event that reservations and wait lists react to.

Timed holds are the mechanism behind "held until payment": inventory returns the reservation with its expiry, a late confirmation is refused, and the unit returns to the pool. **The timeout logic lives in inventory**; the caller only reacts to the result.

**Gaps and the past.** Neither source says whether a slot in the past may be claimed — decide. Say whether the calendar is bounded and what "no record" means (Step 4, blackouts).

**Records into:** the Time Model section.

---

### Step 7: Define Reservations and Demand

*(Reservations: run if level 5 is signalled. Demand shapes: always — every request has a shape.)*

**Reservation.** The published language on top of availability: *an owned bundle of claims with a purpose and a status*. One reservation may hold several claims (a four-night stay). It references availability **only by claim identity** — it stores no domain meaning of its own. The caller (an order, a process) holds only a **reference** to the reservation and may treat its presence as evidence that fulfilment has begun.

**Purpose** is descriptive classification for reporting (customer booking, internal allocation, technical hold); it drives no behaviour unless the modeler binds a hold duration to it, in which case say so.

**Lifecycle.** *This is a design decision:*
- **Born confirmed** — the claims are acquired first; the reservation exists only if all of them succeeded. Simplest; no pending state.
- **Pending first** — the reservation is created, then allocation is attempted; on shortage it stays pending. Choose this when the caller needs a durable handle on demand that is not yet satisfied. The alternative under a wait list or retry policy is to keep queued demand *outside* the reservation model — an entry on the wait list, not a pending reservation — and let the caller hold its own pending state, which is what the sources describe: the order goes to "pending allocation", inventory returns only a result.

Name only the states the domain distinguishes. A minimum is *held* and *released*; add pending, confirmed, cancelled, expired and fulfilled only where the business treats them differently. Transitions worth checking:
- *confirm* — all claims acquired;
- *cancel* — owner-authorized, releases every claim, terminal; after execution has started this is a **compensation** (release resources, refund, stop work), not a status flip;
- *expire* — a timed hold lapsed (Step 6);
- *fulfil* — the resource was taken (Step 9); the claim is released and consumption recorded.

**Partial release.** *Decision:* on a quantity decrease, can inventory reduce a claim's amount or release one claim of a group, or is it cancel-and-reserve-again? *"Inventory knows how to release part of a reservation; the order only knows the sequence."*

**Demand shapes.** A request is expressed in **item-type language, never resource language** — "a diesel tank", "an iPhone", "Dr Kowalski at 09:00" — and exactly one component translates it into claims. Three shapes, matched to availability kinds:

| Demand shape | Says | Resolves to |
|--------------|------|-------------|
| **a specific item** | "that laptop, serial 123" | one claim on the exclusive record mapped to that instance |
| **an amount** | "500 L of diesel", "3 consulting hours" | one claim on a pool; inventory chooses the pool |
| **a set of periods** | "room 101, nights 15–17" | one claim per slot, all-or-nothing |

**Resource choice for unpinned demand.** When the request names a type and several resources qualify, *decision*: first free, nearest, oldest lot first (FIFO/FEFO), load balancing, customer preference, or spread across several pools. Name the rule; the sources name the options but do not design one. Pinning a specific instance is allowed but is a *condition of fulfilment*, and it removes this flexibility.

**Records into:** the Reservations & Demand section.

---

### Step 8: Define Allocation Timing, Shortage Policy and Wait List

*(Allocation timing: always. Shortage policy and wait list: run if shortage handling is signalled — level 6.)*

**Allocation timing.** Usually declared on the product; sometimes it varies per channel or process. Inventory must support every variant:

| Timing | Meaning | What inventory sees |
|--------|---------|---------------------|
| Pre-allocated | the reservation exists before the request (courier slot, limited tickets) | a claim first; later a confirm that checks the hold is still valid |
| At confirmation | nothing held in advance; allocation is attempted when the request is confirmed (doctor's visit) | a request with preferences; inventory decides what is possible |
| External | the reservation lives in a partner's system | nothing — inventory takes no part |
| None | nothing to allocate (e-book) | nothing |

**Rule:** timing and shortage policy are properties of the item type (or the process), not of the caller — one business request may fan out into several allocation requests with different timings and policies: one pre-allocated line, one allocated at confirmation, one queued. Inventory treats them uniformly; only the parameters differ.

**Shortage policy — what happens when there is not enough.** *"Unavailability does not always mean failure."* A product (or process) decision, **passed explicitly with the request, never guessed by inventory**:

| Policy | Meaning | Example |
|--------|---------|---------|
| Reject | no capacity, no contract | airline seats |
| Timed hold | held for a limited time to finish payment (Step 6) | concert tickets |
| Wait list | the request queues; when capacity frees, allocation is re-attempted | specialist visits |
| Retry | capacity appears and disappears; the *caller* schedules retries | cloud resources |
| Expand | capacity is created on demand | storage, account-number pools |

**Rule:** inventory reports the *result* of an allocation attempt; **the caller decides its own state**. Inventory never sets an order's status.

**Wait list.** A queue of pending demand plus a replaceable **selection policy**. The queue does not know availability; it is told what can currently be fulfilled. Being selected and leaving the queue are one act.

| Selection policy | Rule | Use when |
|------------------|------|----------|
| Arrival order | longest waiting first, unconditionally | fairness by time |
| Priority | best priority first, ties by arrival | emergencies before routine |
| First fit | first entry in arrival order that can be fulfilled now | amounts of different sizes against a pool |
| Quota per segment | first fittable entry whose segment still has quota | fairness between customer classes |

**Decisions to record:**
- **Who triggers the re-attempt** — *this is a design decision*: the sources say inventory initiates re-allocation when a resource frees (wait list) but the caller schedules retries (retry policy). Choose per policy and say whether a release, an expiry or a delivery triggers selection.
- Is the selected entry **served** immediately (a claim with the requested duration) or **offered** with a deadline? An offer is a timed claim on the freed resource; confirmation converts it to the requested duration, a lapse releases it and drops or re-queues the entry (decide which), and selection runs again. What happens if serving fails after the entry has left the queue?
- Where the priority or segment used by selection comes from — triage, customer class, a rules module. Inventory applies it and never computes it.
- Is the wait list per resource, per item type, or global? Bounded or unbounded — and what happens to a request that arrives at a full queue? Do entries expire?
- Whether a waiting request auto-confirms the caller's order or needs re-approval.

**Records into:** the Allocation Timing & Shortage section.

---

### Step 9: Define Stock Identity and Consumption

*(Run if stock identity or consumption is signalled — level 7.)*

**Item type.** Inventory holds a local projection of the product definition — identity, name, tracking strategy, preferred unit — and nothing else. Product is upstream: *"inventory receives product definitions, knows their types and instances, tracks the physical side — availability, quantity, location — but does not create definitions."* Inventory is where a unit or feature incompatible with the item type is caught — the sources put that check here rather than in the caller. Whether inventory holds enough of the definition to decide, or calls back to product, is a decision.

**Tracking strategy** — a property of the item type, the stable axis to model on:

| Strategy | Identity of an instance | Examples |
|----------|-------------------------|----------|
| Unique | definition = instance; exactly one exists | an artwork, a collector's item |
| Individually tracked | serial number mandatory | phones, cars, laptops |
| Batch tracked | lot mandatory; units within a lot are alike | milk, medicines, a day's dispatch |
| Serial and batch | both | a TV (warranty and quality control) |
| Interchangeable | quantity only | screws, fuel, flour, electricity |

Mapping to availability kinds (a convention, not enforced by the archetype): unique / serial → exclusive or time-partitioned; interchangeable / batch → pool.

**Instance.** A concrete thing that exists physically or logically, has identity, and can be allocated, reserved or created: item type, optional serial, optional lot, optional quantity, open feature values (colour, size). **Rule:** an instance must be identifiable by a serial and/or a lot unless the type is interchangeable. Each instance counts as at least one unit, so "how much do we hold" is always answerable. Instances of goods come into being at production or goods receipt, before any customer; instances of services come into being when someone books.

**Batch.** Exists as an identifier on instances plus whatever the domain needs — production date, use-by date, quantity in lot. The sources give no recall, split or merge operation; add them only on signal.

**Held ≠ offerable.** Creating an instance does not create availability. The bridge is an explicit, optional mapping *instance → resource*, held in one place per item type. An unmapped instance is stock that cannot be claimed. State who maintains the mapping and when.

**Count vs capacity.** *Decision:* is a pool's capacity to promise **declared** (the tank holds 5,000 L) or **derived** from the summed quantity of mapped instances? The sources model both; pick one per resource, and if both numbers exist say how they are reconciled.

**Consumption.** Reserved is not taken. A pool distinguishes *held* (active claims) from *consumed* (withdrawn from stock): fulfilment releases the claim and records the amount actually taken; the unused remainder returns to the pool. Deliveries replenish. Units are compared for equality only — adding across units is an error; conversion, if needed, is a product-side rule. Which lot a consumption drew from (FIFO, FEFO, manual) is a selection rule (Step 7), decided at write time.

**Records into:** the Stock & Identity section.

---

### Step 10: Define Boundaries and Integration

**Always record the boundaries.** Inventory is upstream of Ordering and downstream of Product; it receives commands and returns results, never domain knowledge, and never depends on its callers.

| Neighbour | Direction | What crosses the boundary | What must not cross |
|-----------|-----------|---------------------------|---------------------|
| **Product** | upstream of inventory | item type, tracking strategy, units, allocation timing, shortage policy, applicability | availability, quantities, which instance |
| **Ordering** | downstream of inventory | an allocation request in resource language with an explicit policy; back: a result and a reservation reference | order status, business reason, pricing |
| **Party** | side | a capability (what a party can do, when, up to what volume), usually the input to slot generation — who turns it into a calendar is a decision the sources leave open; parties as opaque owners | treating capability as availability |
| **Accounting** | side | consumption and intake as bookable movements; reservations as off-balance entries if the business wants them audited | inventory deciding what to book |
| **Pricing** | side | nothing structural; price is validated before allocation is attempted | availability-driven pricing (not in the sources) |

**The contract.** One request carries: resource type, resource identity or item type, time, place, duration, quantity, owner, the policy to apply, any priority or segment for wait-list selection, and a generic context map inventory uses but does not interpret. One result carries: success or failure with reasons, the claim or reservation identities, and any expiry. *"This code does not change by industry; only the inputs, the product strategy, the policy and the resource context change."*

**Anti-patterns to name in the model:**
- the order that grows into a warehouse, logistics and availability system;
- a generic inventory that learns domain concepts and reacts to business events (nine-month features);
- the word "available" used in two senses — scarcity (inventory) and permission (product applicability).

**Contention beyond the calendar (run if level 8 is signalled).** When a free slot is not enough — requests conflict because they share power, cooling, acoustics, a shared balance — availability is a *connected-component* question over an influence graph, and required capacity is a *colouring* question ("how many parallel lanes?"). Keep the calendar in inventory, model the conflicts as a separate graph, and intersect them at admission time. Record this as an integration boundary, not as a bigger reservation model.

**Records into:** the Boundaries & Integration section.

---

### Step 10.4: Write the Worked Walkthrough

Run one concrete sequence through the model **from the tables only**: register capacity, take claims, hit a shortage, release or expire one, and consume one where consumption exists. Carry running on hand / held / free on every row and re-check `free = on hand − held` by hand (for an exclusive resource, held is 0 or 1). Include at least one request the model **refuses** and, where a wait list exists, one selection. If a row cannot be answered from the tables, a layer is missing.

**Records into:** the Worked Walkthrough section.

---

### Step 10.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:
- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred from a rule in this skill (cite the step)
- **(X)** — neither: assumed silently

Walk, in order: every Category A and Category B question from Step 2; every block marked **This is a design decision**, **Decision** or **Convention** in Steps 4–10; every availability kind chosen in Step 4; every state in the reservation lifecycle; every row of the shortage policy. Tag each.

| Decision area | Example decisions to check |
|---------------|---------------------------|
| Availability kind | Exclusive or divisible? Per period? Capacity declared or derived? |
| Ownership | Who may release? Override? Transfer? Re-claim by the same holder? |
| Time | Slot granularity and origin? Hold permanent or timed? Expiry lazy, swept or event? Past slots? |
| Groups | All-or-nothing? Group release atomic? |
| Reservation | Born confirmed or pending? Which states reachable? Partial release? |
| Demand | Which shapes? Who chooses the resource for unpinned demand, by what rule? |
| Shortage | Which policy? Who triggers re-attempt? Serve or offer? Wait list scope and bound? |
| Stock | Tracking strategy? Held vs offerable mapping owner? Consumption tracked? Units and conversion? |
| Boundaries | Where does allocation timing live? Does accounting book movements? Is any conflict indirect? |

**For every (X) decision found:**

1. If the decision has low impact (purely technical, easily changed): mark as explicit assumption in Implementation Notes.
2. If the decision affects business behavior (who may release, what expiry does, whether waiting is possible, whether partial acquisition can stand): **stop and ask** using `AskUserQuestion` before delivering the model. Without `AskUserQuestion`, deliver with the assumption marked **(X — confirm)**.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

```markdown
# Inventory Archetype Model: [Domain Name]

## Complexity Level
[Highest level modeled and the list of signalled layers; skipped layers named]

## Concept Mapping

| Domain Concept | Inventory Archetype | Notes |
|----------------|---------------------|-------|
| ...            | ...                 | ...   |

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
[Question → answer, tagged (R)/(A)/(I)/(X); the Step 10.5 checklist rolls into this section; "It depends" answers listed as externally supplied parameters]

## Resources & Availability

| Resource | Kind | Capacity / partition | Unit | Availability rule | Notes |
|----------|------|----------------------|------|-------------------|-------|
| [name]   | exclusive / divisible / time-partitioned / bundle | [1 for exclusive; on-hand formula for divisible; slot granularity for time-partitioned; components for a bundle] | [unit, or — for a single thing] | [formula or "free unless a live claim exists"] | [blackouts, origin of capacity] |

## Claims & Ownership
[Who may claim, who may release, re-claim rule, group rule (all-or-nothing, rollback), overrides]

## Time Model   (omit if levels 2–3 are not signalled — say so)
[Slot definition and origin; hold duration per purpose; expiry handling; past slots]

## Reservations & Demand   (demand shapes always; omit the reservation half if level 5 is not signalled — say so)
[Demand shapes and resource choice rule; reservation = owner + purpose + claims + status; lifecycle table; partial release]

| State | Entered by | Leaves by | Effect on claims |
|-------|-----------|-----------|------------------|

## Allocation Timing & Shortage   (allocation timing always; omit the shortage half if level 6 is not signalled — say so)
[Allocation timing per item type; shortage policy per item type; wait list scope, bound, selection policy, priority source, trigger, serve-or-offer]

## Stock & Identity   (omit if level 7 is not signalled — say so)
[Item types with tracking strategy and unit; instance identity; batches; instance → resource mapping and its owner; count vs capacity; consumption and replenishment]

## Boundaries & Integration
[Table of neighbours: what crosses, what must not; the request/result contract; any graph-modeled contention]

## Worked Walkthrough
[A dated sequence of claims, releases, expiries and consumption with running on-hand / held / free numbers]

## Implementation Notes
[Key decisions, assumptions for unanswered questions, deliberately not modeled layers, externally supplied parameters]
```

---

## Common Patterns & Pitfalls

### Pattern: Inventory Speaks the Resource Language

Inventory knows resource types, identities, time, place, duration and quantity. The doctor, the courier and the tank exist only at the edge, where the UI or the process translates them. This is what lets one inventory serve banking, healthcare, logistics and cloud without changing its contract. If a rule needs to know *what kind of domain thing* the resource is, it belongs in product or in the process, not in inventory.

### Pattern: Inventory Reports, the Caller Decides

An allocation attempt returns a result — allocated, held until, queued, rejected with reasons. What that means for the order, the visit or the shipment is the caller's decision. Inventory never sets another module's status, and the caller never computes availability. The reservation reference on the caller's side is the whole coupling.

### Pattern: Availability Is Computed, Never Stored

"Is it free?" is a function of the claims on the record, their durations and the clock. An availability flag must be kept in sync with claims, expiries and releases and will drift; a computed answer cannot. The same applies to a pool: free = on hand − Σ active claims, evaluated on read.

### Pattern: Policy Is Passed In, Not Guessed

Hold duration, shortage policy, all-or-nothing, selection rule, allocation timing — all vary by product, channel or process. Inventory receives them with the request and enforces them mechanically. The module that knows the product decides what they are.

### Pitfall: "It's a Document with Statuses"

The business says *"no capacity today, the customer waits until something frees up"* and the model answers with a document and a status field. That is availability and a wait list wearing a disguise. The tell: the same request succeeds at one moment and fails at the next while nothing about the request changed.

### Pitfall: Capability Mistaken for Availability

"Maria can do ultrasound, Mon–Fri, 15 a day" is what Maria *can* do; it generates a calendar. Whether Wednesday 10:00 is free is inventory's answer over that calendar. Modelling the capability as bookable slots, or the calendar as a capability, fuses two modules.

### Pitfall: A Free Slot Is Not Always Enough

Where reservations interfere through shared infrastructure, accepting one may make others unsafe. That is not a reason to grow the reservation model; it is a second graph intersected at admission.

### Pitfall: Instances for Every Possible Slot

Pre-creating a product instance per bookable slot turns a calendar into stock. The slot is a resource; the visit is an instance created after the reservation. If the product type cannot exist without a slot, make the type require one — or an availability claim — at the start of the request.

---

## Quality Checks

Before returning the model, verify:

- [ ] Fit test applied; a level-0 domain produced the ❌ block and no model
- [ ] Complexity level stated with every signalled layer and every skipped layer named
- [ ] Every resource has a kind, a unit and an availability rule; pool formulas name capacity origin and whether consumption exists
- [ ] Every claim rule names the owner check; re-claim and override are decided, not implied
- [ ] Groups are all-or-nothing with rollback stated, or the alternative is explicit
- [ ] Time slots are half-open and one slot = one claim; hold durations and expiry handling are stated per purpose
- [ ] Reservation lifecycle lists reachable states only; cancellation after execution is described as compensation
- [ ] Every demand shape maps to an availability kind; resource choice for unpinned demand is named
- [ ] Shortage policy is explicit per item type and passed in; wait-list trigger and serve-or-offer are decided
- [ ] Tracking strategy and held-vs-offerable mapping are present when stock identity is signalled
- [ ] Boundaries table present; nothing domain-specific crosses into inventory; caller decides its own state
- [ ] Worked walkthrough arithmetic checked by hand; free = on hand − held on every row
- [ ] Concept mapping and Unmapped Concepts present (even if empty)
- [ ] All clarifying answers or assumptions reflected; every (X) decision confirmed or marked

---

## Example

**Input:** "A fuel depot has one diesel tank holding 5,000 L. Fleet companies reserve fuel for their vehicles in advance and must fuel up within 30 minutes, otherwise the reservation lapses and the fuel is free again. Only the company that reserved can cancel. If the tank cannot cover a request, the company may wait; when fuel frees up, waiting requests that fit are served in the order they arrived. A tanker delivery refills the tank."

**Detected level:** 7 — signalled layers: 1 (divisible pool), 3 (timed holds), 5 (reservations with owner and cancellation), 6 (wait list, first fit), 7 (consumption and delivery; interchangeable item type). Not signalled: 2 (no time slots), 4 (no composite demand), 8 (no indirect contention).

**Output:**

```markdown
# Inventory Archetype Model: Fleet Fuel Depot

## Complexity Level
Level 7. Modeled: divisible pool (1), timed holds (3), reservations (5), shortage handling with a
wait list (6), consumption and replenishment (7). Deliberately not modeled: time-partitioned
availability (2), composite demand (4), contention beyond the calendar (8).

## Concept Mapping

| Domain Concept | Inventory Archetype | Notes |
|----------------|---------------------|-------|
| Diesel tank | Resource, divisible pool | one resource, one availability record |
| Litres | Unit | equality only; no conversion |
| Diesel | Item type, interchangeable | quantity only; no serial, no lot |
| Fleet company | Owner | opaque identity; party details stay in Party |
| "reserve fuel" | Reservation with one pool claim | demand shape: an amount |
| "within 30 minutes" | Hold duration, timed | expiry frees the amount |
| "cancel" | Reservation cancel, owner-authorized | releases the claim |
| "fuel up" | Consumption | releases the claim, withdraws the amount taken |
| "may wait" | Shortage policy: wait list | first fit in arrival order |
| Tanker delivery | Replenishment | raises on-hand; triggers wait-list selection |

## Unmapped Concepts
- Vehicles — fuel is reserved "for their vehicles", but inventory tracks only the owning company; per-vehicle allocation would be a caller concern or a second owner level. Invoicing of fuelled litres is a boundary (Accounting), not an unmapped concept.

## Clarifying Questions & Answers
- Exclusive or divisible → divisible (R)
- Hold lifetime → timed, 30 minutes, expiry frees the fuel (R)
- Who may release → holder only (R); operator override not wanted (A)
- Shortage policy → wait list (R); selection = first fit in arrival order (R)
- Reserved vs consumed → yes; fuelled litres are withdrawn, unused remainder returns (R)
- Capacity origin → physical tank capacity 5,000 L bounds on-hand; free is derived from on-hand (A)
- Re-claim by same holder → a second reservation is a separate claim (A)
- Partial release → not needed; a reservation is fuelled once (A)
- Wait-list trigger → inventory re-runs selection on every release, expiry and delivery (A)
- Over-booking → none (A)

## Resources & Availability

| Resource | Kind | Capacity / partition | Unit | Availability rule | Notes |
|----------|------|----------------------|------|-------------------|-------|
| diesel_tank | divisible | on hand ≤ 5,000 L | L | free = on hand − Σ active claim amounts | on hand = 5,000 − fuelled + delivered |

A request is accepted when amount ≤ free; exactly the remainder is accepted; one litre more is refused.

## Claims & Ownership
- Claim = owner (fleet company), amount, taken-at, duration 30 minutes.
- Only the owner releases (cancel) or consumes (fuel up); another company's attempt is refused.
- Several companies hold amounts of the same tank at once; one company may hold several claims.
- No groups (single resource); no override; no transfer.

## Time Model
- No time slots (the tank is not partitioned).
- Hold duration: timed, 30 minutes from the reservation; expired when now ≥ taken-at + 30 min.
- Expiry handling: event — expiry marks the reservation expired, stops counting the amount and
  triggers wait-list selection.

## Reservations & Demand
Demand shape: an amount; only one pool exists, so resource choice is trivial.
Reservation = owner + purpose "fuelling" + one claim + status. Born confirmed (the claim is taken
first).

| State | Entered by | Leaves by | Effect on claims |
|-------|-----------|-----------|------------------|
| confirmed | claim acquired | cancel / expire / fuel up | amount held |
| cancelled | owner cancels | — (terminal) | claim released, free grows |
| expired | 30 minutes elapse unfuelled | — (terminal) | claim released, free grows |
| fulfilled | owner fuels up | — (terminal) | claim released; fuelled litres withdrawn; unused remainder returns |

Pending is not reachable: a request that does not fit goes to the wait list as a *demand*, not as a
reservation.

## Allocation Timing & Shortage
- Allocation timing: at confirmation (the request is the reservation).
- Shortage policy: wait list, passed with the request.
- Wait list: per resource (the tank), unbounded in this model; entry = owner, amount, arrival time.
- Selection policy: first fit — scan in arrival order, take the first entry whose amount ≤ free;
  the selected entry becomes a confirmed reservation with a fresh 30-minute hold.
- Trigger: inventory re-runs selection after every release, expiry and delivery, repeating while
  an entry fits. Served, not offered.

## Stock & Identity
- Item type: diesel, interchangeable, unit L. No instances, serials or lots.
- Capacity to promise is derived from on hand, bounded by the physical 5,000 L.
- Consumption: fuel up withdraws the litres actually taken and releases the whole claim.
- Delivery adds to on hand; a delivery beyond 5,000 L is refused.

## Boundaries & Integration
| Neighbour | Crosses | Must not cross |
|-----------|---------|----------------|
| Product | "diesel" as an interchangeable item type in litres; shortage policy = wait list; hold = 30 min | on-hand amounts |
| Ordering | fuelling request (item type, 1,500 L, owner, policy) → result + reservation reference | order status |
| Party | fleet companies as opaque owners | company data |
| Accounting | fuelled litres and deliveries as movements for invoicing and stock ledger | which claim to release |

## Worked Walkthrough
Tank capacity 5,000 L; holds last 30 minutes; wait list first fit.

| Time | Event | Outcome | On hand | Held | Free |
|------|-------|---------|--------:|-----:|-----:|
| 08:00 | tank registered | — | 5,000 | 0 | 5,000 |
| 08:10 | Fleet A reserves 4,000 (hold to 08:40) | confirmed | 5,000 | 4,000 | 1,000 |
| 08:15 | Fleet B requests 1,500 | 1,000 free → wait list [B 1,500] | 5,000 | 4,000 | 1,000 |
| 08:20 | Taxi Corp reserves 800 (hold to 08:50) | confirmed | 5,000 | 4,800 | 200 |
| 08:25 | Taxi Corp reserves 200 (hold to 08:55) | confirmed — exactly the remainder | 5,000 | 5,000 | 0 |
| 08:30 | Fleet A fuels 3,800 | fulfilled; unused 200 returns; wait list: B does not fit | 1,200 | 1,000 | 200 |
| 08:35 | Fleet C requests 300 | 200 free → wait list [B 1,500, C 300] | 1,200 | 1,000 | 200 |
| 08:45 | Taxi Corp fuels 800 | fulfilled; wait list: nothing fits | 400 | 200 | 200 |
| 08:55 | Taxi Corp's 200 hold lapses | expired; wait list: B no, C yes → C confirmed (hold to 09:25) | 400 | 300 | 100 |
| 09:10 | Fleet C fuels 300 | fulfilled | 100 | 0 | 100 |
| 09:30 | tanker delivers 3,000 | on hand 3,100; wait list: B fits → B confirmed (hold to 10:00) | 3,100 | 1,500 | 1,600 |
| 09:50 | Fleet B fuels 1,500 | fulfilled | 1,600 | 0 | 1,600 |

Check: fuelled 3,800 + 800 + 300 + 1,500 = 6,400; 5,000 + 3,000 − 6,400 = 1,600 on hand; free =
on hand − held on every row.

## Implementation Notes
- Fleet A reserved 4,000 but fuelled 3,800: the claim is released whole and only the litres taken are
  withdrawn — the remainder returns to the pool without a separate release.
- Fleet B waited through two events that did not fit (08:30, 08:45) and one where a later, smaller
  request was served first (08:55): first fit trades arrival fairness for utilisation — chosen (R).
- Hold duration and shortage policy are supplied by the product definition of "diesel"; the depot
  could later run a second item type (petrol) with different values without changing inventory.
- Assumption (X — confirm): a delivery that would exceed 5,000 L is refused rather than partially
  accepted.
- Assumption (X — confirm): a non-empty wait list does not block direct requests — Taxi Corp at 08:20
  and 08:25 took fuel while Fleet B waited, because new requests are not routed through the queue.
- Deliberately not modeled: time slots, composite demand, serial or lot tracking, over-booking,
  indirect contention.
```

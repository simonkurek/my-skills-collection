---
name: ordering-archetype-mapper
description: Transform domain requirements into an Ordering Archetype model — an order as an identifiable request for fulfillment that orchestrates product, pricing, parties, inventory, payment and fulfillment without owning any of them. Identifies complexity level (0–9), lines and specifications, roles at order and line scope, price as a recorded fact, state-gated operations with compensation, allocation and fulfillment integration. Produces an implementable model with explicit concept mapping, unmapped concepts and integration boundaries.
argument-hint: "[domain requirements or feature description]"
---

# Ordering Archetype Mapper

Transform any domain description in which *someone asks for something to be done* into an Ordering model. The order does not need to be a shopping cart — it can be a credit application, a lab test referral, a shipment, a cloud provisioning request, an account opening, a production run, a money transfer. Ordering is *"the mechanism that turns an intention into a request for fulfillment"*: it records what was requested, by whom, for whom, at what agreed terms, and how far execution has come. It calls on product, pricing, party, inventory, accounting and fulfillment; it does not replace them.

**Output goal**: A model that gives the domain one reusable skeleton for every "request → confirm → fulfill" process: a self-sufficient record of intent, explicit roles, a frozen price, a lifecycle whose rules tighten as consequences grow, and clean boundaries toward the modules that do the actual work.

## When to Use

**Use this skill when:**
- A party submits a request that another party must confirm, execute and possibly be paid for — and the same shape recurs across business lines under different names (application, referral, booking, contract, activation)
- Who orders, who pays, who receives and who executes are not always the same party, and the agreed terms must survive later changes to catalog, price list or party data
- Requirements talk about statuses, changes after placement, cancellations, partial fulfillment, or things that "should have happened but did not"

**Output is useful for:** domain modeling sessions before implementation, and deciding where ordering ends and product, pricing, inventory, party, accounting or fulfillment begin

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the ordering archetype does not fit, and briefly explain why.

### The core question

> *"Does someone submit an intent that must be confirmed and then fulfilled — and does the business need to know what was requested, by whom, for whom, on what agreed terms, and what actually happened?"*

If **yes** → ordering likely fits.
Counter-questions that route elsewhere:
- *"How much X does S have?"* → accounting · *"What should this cost?"* → pricing · *"Is it available?"* → inventory · *"Who is this party?"* → party (see the ❌ rows below)
- *"What may be ordered, with which variants and features?"* → product & catalog
- *"Did execution match the plan?"* → plan vs. execution
- *"A document moves through statuses"* with no product, no value, no roles → workflow or state machine, not ordering

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "customer places / submits / applies for / books / commissions X" | ✅ Yes — an intent to be fulfilled |
| "who pays is not who orders", "delivered to someone else", "subcontractor executes" | ✅ Yes — roles in an order |
| "they pay the price agreed at the time, not today's price" | ✅ Yes — price as a recorded fact |
| "options chosen per request: variant, slot, parameters, preferences" | ✅ Yes — line specification |
| "draft → confirmed → in progress → done / cancelled, with different rules at each step" | ✅ Yes — order lifecycle |
| "customer changed the address / quantity after placing", "we must know what was originally ordered" | ✅ Yes — change after placement |
| "five business lines, each with its own 'application' code" | ✅ Yes — one skeleton, many processes |
| "how much quota / how many points does the user have" | ❌ No — accounting |
| "compute the tariff from usage, tier and season" | ❌ No — pricing (ordering only records the result) |
| "is the doctor free at 10:00", "how many seats are left" | ❌ No — inventory / availability |
| "ticket goes open → assigned → resolved", no product, no payer, no executor | ❌ No — workflow / state machine |
| "application with statuses" | ⚠️ Borderline — see below |
| "reservation" | ⚠️ Borderline — see below |

**Language signals.** You will rarely hear the word "order". Listen for the verbs and map them: *submit an application / place a request* → draft · *confirm, sign, approve* → confirmed · *in progress / completed* → processing / closed · *perform, deliver, execute* → fulfillment · *withdraw, resign* → cancelled. If those verbs appear alongside diverging statuses, `isConfirmed` / `isActive` / `isCancelled` flags, logic scattered over several services, no single place showing what actually happened, or "many processes, each of them uniquely specific" — the domain is *"implementing Ordering without Ordering."*

### Borderline cases — how to decide

- **"It's just a document with statuses."** Ask what the business actually says about it: is there a *product* (something with operational consequences), a *price or value* (money, points, a budget, a limit), *parties* in different roles, *limited capacity*, something someone *formally executes*? If most of these are present, the "document" is an order and a document workflow is *"a very expensive way of not hearing what the business wanted."* If none are, it is a workflow.
- **Reservation.** An order may *request* a reservation but never *owns* one. A voucher bought today with a date chosen next week is an order without a reservation; a visit at 10:00 with Dr Smith is an order that cannot exist without one. Both are ordering; availability itself is inventory.
- **Plan vs. execution.** If the requirement is mainly comparing what was scheduled with what happened, ordering supplies the intent side only; route the comparison to plan-vs-execution.

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The ordering archetype requires an intent that is submitted, confirmed and fulfilled, with
parties in roles and agreed terms that must be recorded. This domain is a [state machine /
ledger / pricing rule / availability problem / ...] because:

- [specific reason from the requirements]
- The natural question is "[the counter-question that fits]", not "what was requested, by whom,
  on what terms, and what happened?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — who asks for what, who confirms and executes it, who pays, and what can change or fail along the way?"

---

### Step 1: Assess Complexity Level

Identify **every layer that has a signal** in the requirements. Level 1 is the foundation; levels 2–9 are largely independent above it — a domain may need roles without allocation, or allocation without pricing. Model each layer with a signal, **and no further**. Report the level as the highest one modeled, but adopt layers by signal, not by number.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 0 | **Single-shot request** | One thing, one party, done or not done; no price, no roles, nothing attaches to it later | (none — a command or a status field is adequate) |
| 1 | **Identified intent with a lifecycle** | The request is created, confirmed, executed, closed or cancelled; payments, reservations, complaints attach to it; rules differ by state | `isConfirmed` / `isCancelled` flags; status logic scattered across services |
| 2 | **Composable scope** | One request bundles several products, services or options; the offer changes faster than the model | a boolean per option, a column per variant, a migration per campaign |
| 3 | **Semantic scope** | *How, where, when* to execute: variants, a specific instance, creation parameters, a time slot, preferences; conditions under which the request was made (channel, segment, campaign) | notes, e-mails, phone calls, parameters hard-coded in the process |
| 4 | **Roles** | Who orders ≠ who pays ≠ who receives ≠ who executes; parties without accounts; party data changes after placement | a single `userId`; one column per role |
| 5 | **Recorded price** | Price frozen at agreement; negotiated or manual prices; price not known at placement; a breakdown needed for audit, tax or accounting | reading the live catalog price; `total = null` or `0` for "not yet" |
| 6 | **Change after placement** | Changes to a placed request; the business must know what was ordered, what changed, when and why; the cost of a change grows with progress | overwrite in place; cancel-and-recreate that loses history |
| 7 | **Resource allocation** | Some lines compete for limited stock, slots or capacity; a reservation precedes, follows or lies outside the order; shortage means waiting, not failure | the order owns availability logic; every order blocks a resource |
| 8 | **External processes** | Fulfillment, payment or billing run in other modules or teams, take time, fail, and report back | one order class that calls every API and knows every fulfillment variant |
| 9 | **Many process families** | Several fundamentally different order processes in one organization (mortgage, FX deal, term deposit) | one shared "Order" library carrying everyone's fields |

**Guidance — which steps to run:**
- Level 0: the archetype is overkill. Emit the ❌ Does Not Fit block from "When NOT to Use" and stop; do not produce a model.
- Always: Steps 0–3, Step 4 (order, lines, lifecycle), the Boundaries part of Step 10, and Steps 11–11.5.
- Then run one step per **signalled** layer, regardless of the numbers in between: semantic scope → Step 5; roles → Step 6; recorded price → Step 7; change after placement → Step 8; resource allocation → Step 9; external processes → the fulfillment, payment and orchestration parts of Step 10; many process families → the landscape part of Step 10. Composable scope is handled inside Step 4 (lines instead of flags).
- For level 0 say instead: "one thing, one party, nothing attaches to it later — a command and a status field are adequate; the archetype would add ceremony without buying anything."

Partial adoption is normal — the archetype is *"a base, a skeleton; adapt statuses, flows and rules to your needs."* Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps between the requirements and the ordering archetype. Ask about **two categories** in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more are needed).

#### Category A — Standard Ordering decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as **design choices**:

- **Confirmation contract**: What does confirming mean here — price becomes binding, resources are allocated, payment is taken, fulfillment starts — all of these, or a subset?
- **Price at confirmation**: Must every line have a definitive price before confirmation, or may pricing be deferred (estimates, usage-based billing)?
- **Changes after confirmation**: Which changes are allowed once confirmed — none (cancel and reorder), limited ones with compensation, or any?
- **Cancellation**: From which states can the request be cancelled, and what must be undone (release, refund, stop execution)? After fulfillment, is it a cancellation or a return / complaint?
- **Party data**: Reference registered parties only, keep an immutable snapshot of their data at placement, or both? Are unregistered parties allowed?
- **Allocation timing**: Is capacity reserved before the request exists, at confirmation, in an external system, or never? Per product or per channel?
- **Shortage policy**: When capacity is missing — reject, hold with an expiry, queue and wait, retry later, or expand capacity?
- **Payment model**: Pay at confirmation, record a charge for later billing, or let billing react to order events?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the ordering archetype supports but the requirements do not mention**. For each gap found, ask whether that dimension is wanted. Do not limit yourself to this list — reason freely:

- **Line-scope roles**: Can a single line have its own receiver, delivery contact or installation recipient, different from the order's?
- **Role cardinalities**: Exactly one orderer? Payer optional? Several executors?
- **Role eligibility**: Must an executor satisfy requirements (certificate, region, capacity) before confirmation?
- **Manual prices**: May an operator set a negotiated price, and does it need a reason and an author?
- **Discount vouchers**: Are marketing discounts automatic (inside pricing) or explicit items on the order (a negative line)?
- **Execution preferences**: Delivery window, warehouse, gift wrap — do they affect *what* is ordered or only *how* it is executed?
- **Fulfillment failure**: When execution fails, does the order retry, cancel with refund, or wait for a human?
- **Partial fulfillment**: Do lines complete independently, and what does "fulfilled" mean for the whole order?

Collect answers before proceeding. If the user cannot answer, document the assumption made in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

Always include **"To zależy / It depends"** as an explicit option in every `AskUserQuestion` call — do not rely on the automatic "Other" fallback. Place it as the last option in each question. If the user selects it, treat it as a **policy supplied from outside the order**:

- Name the *parameter* the order receives (allocation strategy, shortage policy, role policy, payment model, cancellation window)
- Note in **Implementation Notes** which module decides it — most often the product definition, sometimes the channel or a rules module — and that the order only applies it
- Do **not** model the decision logic inside the order

This is the intended outcome: *"the order does not guess and does not add exceptions; the product definition tells it what to expect."*

#### If `AskUserQuestion` is not available

Do not stop. For every question you would have asked, pick the option most consistent with the requirements, mark it **(X — confirm)** in the Clarifying Questions & Answers section, and continue. Defaults if the requirements are silent: confirmation = allocate → pay → hand off; definitive price required; no changes after confirmation (cancel and reorder); cancellation allowed until execution starts; reference-only party data; allocation at confirmation; shortage = reject; pay at confirmation; direct orchestration. Step 11.5 applies the same policy — it does not add a second one.

---

### Step 3: Map Domain Concepts to Ordering Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Ordering Archetype | Notes                          |
|----------------------|--------------------|--------------------------------|
| [domain noun/verb]   | Order / Order Line / Line Specification / Business Context / Party in Role / Party Snapshot / Line Pricing / Order Status / Fulfillment Status / Allocation Strategy / Shortage Policy / Integration Port | [why] |
```

Nouns that belong to another archetype are mapped to that archetype's boundary, not to ordering (*"catalog"* → product; *"tariff"* → pricing; *"stock"* → inventory; *"balance"* → accounting; *"shipment tracking"* → fulfillment).

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear ordering archetype equivalent:
- [concept] — [reason it doesn't fit / decision needed / which module owns it]
```

This section must be present even if empty (`None identified`). Concepts owned by a neighbour reappear in Boundaries & Integration (Step 10); the rest go to Implementation Notes.

---

### Step 4: Define the Order, Its Lines and Its Lifecycle

**The order** is an identifiable record of intent, findable and referable *"after a month, after five years"*, because everything else attaches to it: payment, reservations, fulfillment, complaints, audit. **Order lines** are the quanta of intent — *this product, in this variant, in this quantity* — each identifiable so it can be added, changed and removed on its own.

| Concept | Meaning | Key rules |
|---------|---------|-----------|
| Order | The request as a whole; anchor for everything that follows | One identifier; a status; a version (e.g. optimistic locking) because users, payment, inventory and fulfillment all write to it. **Decision:** may a draft hold zero lines, and does confirmation require at least one? |
| Order Line | One product type × quantity (+ specification, own price, optional local roles) | References the product **type**, never an instance; quantity in any unit — validation of unit vs. product belongs to catalog or inventory, not here |
| Package line | A bundle sold as one product (laptop + mouse + bag; account + card + e-banking) | Still **one line**; the scope of execution comes from the package definition, not from the line structure |

**Anything that is requested, executed and possibly paid for is a product** — an installation, a training day, a legal action, an administrative decision, even a notification (*"if we treat notifications as a product, which is often true in financial institutions"*). Model it as a product type with its own line, not as a side effect of another line or a flag on the order.

**Rule — lines, not flags.** Options, add-ons and variants are lines or line specifications, never boolean columns on the order. *"A new combination must not require a change to the data model; offer rules must not be baked into the process."*

**Rule — type, not instance.** A line says *"fulfill this product under the applicable rules"*, not *"take this item"*. Products that do not exist before the order (an account, a policy, a subscription) have no instance to point at; bulk products (screws, fuel) have no instance worth naming; and naming the instance removes the fulfillment side's freedom to choose the best warehouse or batch. A specific instance, when the customer insists on it, is a *condition of execution* recorded in the specification (Step 5).

**The lifecycle.** The order is a state machine — *"in some businesses very simple, in others complex"* — whether implemented as a rich object or as a workflow *"does not matter as long as you are aware of the boundaries and the consequences."* Use the following as the **example baseline** and adapt it; name every state the domain needs and none it does not:

```
draft ──confirm──▶ confirmed ──execution starts──▶ processing ──execution done──▶ fulfilled ──▶ closed
  │                    │  (if capacity missing: pending allocation ──capacity freed──▶ confirmed)
  └── cancel ──────────┴── cancel (with compensation) ──▶ cancelled
```

- **draft** — being composed; edits are cheap; only pricing is affected
- **confirmed** — the price is a contract, resources are held, obligations exist; edits are compensation processes
- **pending allocation** — waiting for capacity (only if the shortage policy allows waiting, Step 9)
- **processing** — execution in motion elsewhere; structural edits are usually no longer edits but returns or complaints
- **fulfilled / closed** — done; the remaining work is closing formalities (invoicing, settlement, archiving)
- **cancelled** — terminal; how much had to be undone depends on the state it was left from

*"The further along the lifecycle, the less freedom and the greater the consequence of a single decision. This is not a technical limitation — it is a conscious architectural choice. That is exactly how the world outside the system works."* The operation × state rules are written in Step 8.

**Records into:** the Order & Lines and Lifecycle sections of the Output Format.

---

### Step 5: Define Line Specification and Business Context

*(Run if the semantic-scope layer is signalled — level 3.)*

**Line specification** refines a line without changing what is ordered: *"still the same order, still the same line; only the scope and the manner of execution change."* It is a generic key/value structure, but **its content is not arbitrary — its meaning comes from the product definition.** The product type says which features are mandatory, which optional, which values are allowed, whether the product exists beforehand, is created on demand, is temporal, or is a package. Ordering reads that contract; it does not invent it. Ordering validates the specification against it at placement or at confirmation, not on every keystroke.

Decide the specification's shape per line with these questions, in order:

| Question | If yes, the specification holds | Example |
|----------|--------------------------------|---------|
| Do I mean a specific instance? | An instance identifier (serial number, VIN, account number) | "exactly the phone I saw in the shop" |
| Does the product exist before the order? | Search criteria: feature values, variant, preferences | "blue, 256 GB, any unit" |
| Is it created on demand? | Creation parameters — all mandatory features of the product | "currency PLN, package standard, opening deposit 500" |
| Is it a package? | The component choices and the features of each chosen component | "laptop model X in black; no bag" |
| Is it temporal (time is the resource)? | The resource and the time slot; the instance appears only after execution or after a reservation | "Dr Smith, 2026-01-15 10:00, 30 min" |
| Is it only an execution preference? | A preference, kept apart from features by convention; it changes *how*, not *what*. A named *person* is a role (Step 6), not a preference | "deliver from the Kraków warehouse, gift-wrap" |

**Convention:** keep features, component choices and execution preferences distinguishable (separate maps, a prefix, a typed structure — any works). In complex catalogs *"go one step further and record the execution mode explicitly in the product archetype."*

**Business context** describes the conditions under which the intent arose: time (order date, season), geography (country, region), channel (online, mobile, branch, call centre), customer characteristics (segment, age, employment status), campaign, partner, referral. The order **collects and carries** this context and hands it to the product module to check applicability (*"a VIP account only for VIP-segment customers in a branch or e-banking"*) and to pricing (Step 7). **Rule:** the order never interprets the context.

**Records into:** the Line Specification & Business Context section.

---

### Step 6: Define Parties and Roles

*(Run if the roles layer is signalled — level 4.)*

*"An order is an operational contract between parties: who initiates, who bears the cost, who receives the result, who executes, who formally accepts payment."* These are **roles, not people** — one party may hold several, and the number of parties differs between orders. Model a party–role–order relation, never a column per role.

**Base vocabulary** (extend or narrow it for the domain; the names below are examples, not a fixed set):

| Side | Role | Meaning |
|------|------|---------|
| Ordering side | orderer | initiates the request |
| | payer | bears the cost — not always the orderer (a promotion paid by marketing, a company paying for an employee) |
| | receiver | receives the product or service |
| Executing side | executor | contractually responsible for fulfillment |
| | payment receiver | formally accepts payment — not always the executor (a central settlement unit) |
| Local | delivery contact, installation recipient, pickup authorized, carrier… | roles whose meaning appears only for a specific product or line |

**Scope.** Some roles must be consistent for the whole order (orderer, payer, executor — *"it rarely makes sense for each line to have a different orderer"*); others differ naturally per line (receiver, delivery contact, installation recipient). A line **may** carry its own roles; if it carries none, it inherits the order's. Where it carries some, each named role **overrides** the order's holder of that role for that line; unnamed roles still inherit. Give the order one place that answers *"which roles apply to this line?"* so the merge logic is not scattered.

**Role policy.** A policy checks the set of roles at each scope: which roles are allowed there, and how many holders each may have. **Convention (example policy):** order scope — exactly one orderer, exactly one executor, at most one payer, any number of receivers; line scope — no orderer, payer or executor; any number of receivers, delivery contacts, installation recipients. The trainers name it as configurable (*"perhaps one supplier but two payers"*) — set the cardinalities from the requirements and mark them (R)/(A)/(X).

**Party identity — this is a design decision:**
- **Reference only** (party id + roles) suffices when every participant is registered and the data needed for fulfillment never changes or does not matter.
- **Snapshot** — an immutable copy of the minimum data needed for fulfillment, contact, audit and complaints — is needed when data changes over time (*"Ms Świątek orders this week and is Ms Świątek-Piątek next week; the order still concerns the first name"*) or when participants are not registered (sending a parcel from a locker; a recipient who does not know a courier is coming). The snapshot never updates when the party master changes.
- **Unregistered parties** are allowed only as a conscious business decision: without an id, order history per party and cross-order identification become harder and less trustworthy (aggregating by e-mail or phone need not be reliable) — complicated, not impossible.

A role the policy requires but the requirements never name (the seller as executor, a settlement unit as payment receiver) is inferred and tagged (X — confirm). **Where role assignments come from** is a process decision the model must *enable*, not hide: the orderer usually comes with the command (or the security context); the executor may be chosen by the user, defaulted per product, region or channel, or routed by rules; the payment receiver is often a fixed unit.

**Role requirements.** Holding a role is not the same as being eligible for it (*"an executor of medical equipment deliveries must meet formal, operational and geographic conditions"*). The order receives the requirements for a role in this context from outside, checks them against the party's capabilities (party archetype), and refuses the order — at placement or at confirmation, a decision to record — when they are not met. It does not know what a certificate is.

**Records into:** the Parties & Roles section.

---

### Step 7: Define Pricing as a Recorded Fact

*(Run if the recorded-price layer is signalled — level 5.)*

*"A price in an order must not be a read of the current product price. It must be a snapshot: a record of how and why the price was set."* The customer who ordered on Monday at 5 000 pays 5 000 on Wednesday even though Tuesday's promotion made it 4 500.

**Boundary.** The order does not interpret pricing rules (pricing does) and does not recompute values (it records them). Whether the product knows its price list or pricing knows products is a decision the trainers left open — they chose the former for their example. It gathers the **pricing context** — product, quantity, specification, parties in their roles, business context, pricing time — calls pricing, and stores the result as a fact. Per line or aggregated per call is an integration choice, not a modeling one.

**What a line records:**

| Element | Meaning | Key rules |
|---------|---------|-----------|
| Unit price and total price | Both, as returned | The order never derives one from the other — it does not know rounding, currency or tax rules |
| Breakdown | Named components (base, discount, tax, margin, delivery…) in *total* interpretation, possibly nested | The history of the pricing decision; totals are the only interpretation that sums safely |
| Source | How the price came to be — see below | Expressed as a type, *"not as a comment or a flag"* |
| Definitive? | Whether the price may be invoiced / paid | Distinguishes contractual prices from information |
| Value unit | Whatever pricing returned — a currency, points, a budget, a limit | The order *"assumes nothing at all about value"* |

**Price sources** (a closed set for the domain; add or drop as needed):

| Source | Meaning | Definitive? |
|--------|---------|-------------|
| Calculated | Pricing applied its rules; full breakdown | yes |
| Arbitrary | Set by a person after negotiation or as a correction; carries who, when and why | yes |
| Estimated | For an offer or a preliminary decision; may change or need approval | no |
| Not yet priced | The line exists, the price will come later (usage-based billing; "we don't know how much wood the sculptor will use") | no — reading it is an error, not zero |

**Rules:**
- The order total is the sum of line totals. An unpriced line means "no definitive total yet", never zero — reading it must be an error, not a `0`. **Decision:** does the order expose a provisional total over the priced lines (clearly labelled as such), or no total at all until every line is priced?
- A structural change to a line (quantity, specification) invalidates its recorded price; the line is re-priced, not silently kept.
- After confirmation the price is a contract. A later change is a new, recorded pricing decision with its author and reason — not an overwrite.
- A manual price is *"an explicit business decision with an audit trail, not a field overwrite."*

**Design decision — breakdown of arbitrary prices:** either pricing offers a fixed-price tariff that takes the agreed amount as a parameter and returns a proper breakdown, or the person setting the price supplies the breakdown, or a default breakdown is applied. Choose and record it.

**Discounts — two options, choose per discount kind:**
- **Automatic** ("more than 20 pallets → 15 %") → a component inside pricing's breakdown; invisible as a separate item on the order.
- **Explicit** (vouchers, coupons, an employee's goodwill discount, order-wide discounts) → a **separate line** with a negative price and its own breakdown, aggregated like any product. Clear for the customer, simple audit, natural rules ("no two vouchers"), no pricing change per campaign.

**Records into:** the Pricing section.

---

### Step 8: Define Operations by State and Their Compensation

*(Run if the change-after-placement layer is signalled — level 6.)*

*"The order decides whether an operation may happen at all; it protects invariants atomically. At the same time it starts processes outside its own world — and there the single transaction ends."* Fill an operation × state table: for every operation the domain needs and every state from Step 4, say whether it is allowed, which external effects it triggers, and what must be compensated.

**Guiding rules:**
- The cost gradient of Step 4 is the rule: cheap in draft (only re-pricing), a compensating process in confirmed (adding a line = a new allocation that may fail and a new payment; removing one = release and partial refund; changing quantity = both), a different process entirely in processing (return, complaint, refund). The order knows the **order of steps, not their implementation**. *"Sometimes it is cheaper to place a new order"* — refusing the change is a valid rule.
- **Confirmation** is the moment almost every process meets: price becomes binding, resources are held, payment or billing starts, fulfillment starts, notifications go out. Order the steps; if one fails, the earlier ones are undone (compensating actions) and the order stays where it was. This is the place for sagas, outbox, idempotent retries and observability.
- **Cancellation** is a state change in draft and a compensation process later: release resources, refund, stop execution, inform. After fulfillment it is normally not a cancellation at all.
- **Named boundaries win over baseline states.** When the requirements draw a line the baseline lifecycle does not ("before picking starts", "once shipped"), add or split a state so the rule has a home. Never stretch a rule across a gap the requirements left silent — fill the gap and tag it (X — confirm).
- **History:** changes after placement never overwrite the original intent. The model must answer *"what was ordered, what changed, when and why"* — by versions, an event log or change records; the mechanism is open.

**Table shape:**

| Operation | draft | confirmed | pending allocation | processing | fulfilled | closed / cancelled |
|-----------|-------|-----------|--------------------|------------|-----------|--------------------|
| add line | allowed; re-price | [rule + effects + compensation] | … | … | … | no |
| remove line | … | … | … | … | … | no |
| change quantity / specification | … | … | … | … | … | no |
| set manual price | … | … | … | … | … | no |
| confirm | steps in order | no | no | no | no | no |
| cancel | set status | compensation: … | … | … | return / complaint flow | no |
| fulfillment report | no | accept | accept | accept | accept (close) | no |

Tag every non-obvious cell (R)/(A)/(I)/(X).

**Records into:** the Operations by State section.

---

### Step 9: Define Resource Allocation

*(Run if the allocation layer is signalled — level 7.)*

*"The order describes intent; the reservation describes resource availability in time. An order may, but need not, imply a reservation."* The order requests allocation, records the reference it gets back, and reacts to the result; it never decides availability. Inventory speaks a generic language — resource type, resource id, time, place, duration, quantity — and the specification (Step 5) is the carrier of the allocation intent, translated from the domain by the UI or the process.

**Allocation strategy — when allocation happens.** Usually declared on the product; sometimes it varies per channel or process. Mixing strategies across lines of one order is normal.

| Strategy | Meaning | What the order does |
|----------|---------|---------------------|
| Pre-allocated | A reservation must exist before the order (courier slot, limited tickets) | Holds the reservation reference in the line specification; at confirmation verifies it still holds and commits it — *"you have a reservation or you have no order"* |
| Order-driven | Allocation happens at confirmation (doctor visit, account number) | Sends preferences; on success enriches the specification with what was allocated |
| External | Reservation lives in a partner's system | Handles payment or the contract; may or may not exist as an order |
| None | Nothing to allocate (e-book, generic service) | Skips allocation; still validates and prices |

**Shortage policy — what happens when capacity is missing.** Also a product (or process) decision, passed to inventory explicitly, never guessed:

| Policy | Meaning | Order state afterwards |
|--------|---------|------------------------|
| Reject | No capacity, no contract (airline seats) | confirmation fails, order stays in draft |
| Hold with expiry | Resource held for a limited time to finish payment (concert tickets) | must confirm before expiry; otherwise inventory refuses |
| Waitlist | The request queues; when capacity frees, allocation is retried (specialist visits) | pending allocation → confirmed automatically or after re-approval |
| Retry | Capacity appears and disappears (cloud); the order schedules the retry | pending allocation |
| Expand | Capacity can be created on demand (storage, account-number pools) | allocated |

**Rule:** inventory reports the *result* of an allocation attempt; **the order decides its own state**. Record for each strategy where the reservation or allocation reference is kept, what a partial allocation means for the order, and what releases the resource on cancellation.

**Records into:** the Allocation section.

---

### Step 10: Define Fulfillment, Payment and Boundaries

*(Fulfillment, payment and orchestration parts: run if the external-processes layer is signalled — level 8. Landscape part: many process families — level 9. Boundaries: always.)*

**Fulfillment status.** The order does not perform fulfillment, but it must know what happens to it — to move to fulfilled, to know when it can be closed, to react to failure, to inform the customer. So it keeps a **fulfillment status of the order as a whole** (not started, in progress, partially completed, completed, failed — adapt), separate from the order status, reported from outside, and mapped onto the lifecycle. If lines complete independently, the order's status is an aggregate of line statuses and the model must say what "fulfilled" means for the whole. For the order, fulfillment is always the same signal; for the operational system it is shipping, account activation, sending a link, performing a massage, an SMS. *"The order does not know how; it knows whether, when and why."*

**Payment and billing — choose per domain (this is a design decision):**
- **Pay at confirmation** — "no money, no order"; the payment result gates the state
- **Billing record** — B2B, net terms: the order registers a charge with billing and continues
- **Billing by events** — where many things influence billing, the order publishes events and billing reacts; the order listens for payment events to advance

**Orchestration style — three stages the trainers describe; pick the one matching the organization:**

| Style | The order… | Fits when | Cost |
|-------|-----------|-----------|------|
| Direct | knows what to do with each line and calls the modules in sequence | one dominant process, a small stable system | the order accumulates operational knowledge |
| Router + handlers | delegates each line to a handler chosen by product type | the offer grows; new product = new handler, not a change to the order | a new contract to govern; registry, failures, harder debugging |
| Announce and react | ends its responsibility at confirmation, publishes the fact; fulfillment modules react and report back by events or commands | fulfillment belongs to several teams / bounded contexts; long-running processes | no single place shows the whole process; eventual consistency, correlation ids, tracing |

In the announce-and-react style, say how progress comes back: either fulfillment publishes its own events and the order subscribes — process knowledge creeps back into the order through the contract of every event it must understand — or fulfillment sends explicit commands to ordering, and each handler must know when and how to report. *"Coupling is not eliminated; it is moved and dispersed."* Correlation ids and distributed tracing are then not optional. Commands versus events are logical patterns, not a synchronous/asynchronous decision.

**Boundaries — always.** Ordering is **downstream** of product, pricing, inventory, party and accounting: it sends them commands and queries and may listen to their events; they never need to know ordering exists. Restate what each neighbour owns and what the order keeps:

| Neighbour | The order asks it for | The order keeps |
|-----------|-----------------------|-----------------|
| Product & catalog | what may be ordered, which features are mandatory or allowed, execution mode, tracking strategy (unique / individually tracked / batch / fungible), allocation strategy, applicability | product type id, specification |
| Pricing | the price for a context | unit, total, breakdown, source, value unit |
| Party | who is who, capabilities for role requirements | party id and/or snapshot, roles |
| Inventory | availability, reservation, allocation | reservation / allocation references, result |
| Accounting / payment / billing | transfers of value | payment or billing references, status |
| Fulfillment | execution | fulfillment status, references |
| Rules | complex branching decisions along the process | the decision outcome |

**Landscape** *(level 9 only)*. With several fundamentally different order processes, no single ordering is ideal; name the trade-off chosen:

| Option | Gain | Price |
|--------|------|-------|
| Shared library core | no duplication | everyone's fields in one model; release waves; binary coupling |
| Separate ordering per business | team autonomy, independent deploys | the same bug in three places; cross-cutting reports need integration |
| Central platform + plugins | standardisation, one flow | the extension API becomes the bottleneck; business knowledge leaks into the centre |
| Separate orderings behind one facade | one customer-facing API, autonomy inside | the facade must know all implementations or a registry; risk of a leaking abstraction |

**Records into:** the Fulfillment & Payment, Boundaries & Integration, and (level 9) Landscape sections.

---

### Step 11: Write the Worked Walkthrough

Run one concrete request through the model **from the tables only**: place it, price it, confirm it, report fulfillment, and then one deviation — a change after confirmation, a shortage, a cancellation or a failed step. At each step name the state, the external effect, the recorded values (price, roles, references) and the concept that decides it. Include at least one operation the model **refuses** and one question the model must route to another archetype (availability, the tariff rule, the balance). If a step cannot be answered from the tables, a layer is missing or under-specified — go back.

---

### Step 11.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:
- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred from a rule in this skill (cite the step)
- **(X)** — neither: assumed silently

Walk, in order: every Category A and Category B question from Step 2; every block marked **This is a design decision**, **Decision** or **Convention** in Steps 4–10; every state in the lifecycle; and every cell of the operation × state table. Tag each.

**For every (X) decision found:**

1. If the decision has low impact (purely technical, easily changed): mark as explicit assumption in Implementation Notes.
2. If the decision affects business behavior (what confirmation commits to, what cancellation undoes, whether waiting is possible, who may pay): **stop and ask** using `AskUserQuestion` before delivering the model. Without `AskUserQuestion`, deliver with the assumption marked **(X — confirm)**.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

Use `—` for a cell that does not apply and *not specified* for one the requirements leave open (and tag it in Step 11.5).

```markdown
# Ordering Archetype Model: [Domain Name]

## Complexity Level
Level [n] — [name]. Layers modeled: [list]. Deliberately not modeled: [list].

## Concept Mapping

| Domain Concept | Ordering Archetype | Notes |
|----------------|--------------------|-------|
| ...            | ...                | ...   |

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
- [question] → [answer or "(assumed) …"]

## Order & Lines

Order: [what it represents; identifier; version]

| Line | Product type | Quantity | Package? | Notes |
|------|--------------|----------|----------|-------|

## Lifecycle

| State | Meaning | Terminal? | Entered by |
|-------|---------|-----------|------------|

## Line Specification & Business Context   (omit if the layer is not signalled — list it under Deliberately not modeled)

| Line | Specification shape | Keys (from product definition) | Execution preferences |
|------|---------------------|--------------------------------|------------------------|

Business context: [keys carried; who interprets them]

## Parties & Roles   (omit if the layer is not signalled — list it under Deliberately not modeled)

Party data: [reference / snapshot / both]; unregistered parties: [allowed? / no]

| Role | Scope | Cardinality | Source of assignment | Requirements |
|------|-------|-------------|----------------------|--------------|

Instances: [which party holds which roles, at order and line scope]

## Pricing   (omit if the layer is not signalled — list it under Deliberately not modeled)

Sources allowed: [list]; definitive price required at confirmation: [yes / no / per line]
Discount style: [automatic / explicit line / both]

| Line | Source | Unit | Total | Value unit | Breakdown | Definitive? |
|------|--------|------|-------|------------|-----------|-------------|

Order total: [sum or "none until every line is priced"]

## Operations by State   (omit if the layer is not signalled — list it under Deliberately not modeled)

| Operation | draft | confirmed | pending allocation | processing | fulfilled | closed / cancelled |
|-----------|-------|-----------|--------------------|------------|-----------|--------------------|

Confirmation steps in order: [1 … n, with what is compensated on failure]
History mechanism: [versions / events / change records]

## Allocation   (omit if the layer is not signalled — list it under Deliberately not modeled)

| Line | Strategy | Shortage policy | Reference kept | Released on cancel by |
|------|----------|-----------------|----------------|-----------------------|

## Fulfillment & Payment   (omit if the layer is not signalled — list it under Deliberately not modeled)

Fulfillment status vocabulary: [list] → order status mapping: [table or list]
Aggregation across lines: [rule]; on failure: [reaction]
Payment model: [pay at confirmation / billing record / billing by events]
Orchestration style: [direct / router + handlers / announce and react] — [reason]

## Boundaries & Integration

| Neighbour | The order asks it for | The order keeps |
|-----------|-----------------------|-----------------|

## Landscape   (omit if the layer is not signalled — list it under Deliberately not modeled)
[option chosen and the price paid]

## Worked Walkthrough
[one request through the model plus one deviation; each step: state, effect, recorded values, deciding concept]

## Implementation Notes
[Key decisions, assumptions for unanswered questions, skipped layers, policies supplied from outside]
```

---

## Common Patterns & Pitfalls

### Pattern: The Order Coordinates but Does Not Execute

The order knows *whether* something must happen, *when* and *why*; product, pricing, inventory, accounting and fulfillment know *how*. A rule about tariffs, stock or eligibility inside the order model has leaked — put it back and let the order carry only the result. Change the courier or the payment gateway, and *"the order does not even notice."*

### Pattern: Policies Come from Outside, the Order Applies Them

Allocation strategy, shortage policy, role cardinalities, role requirements, payment model, cancellation windows — each is supplied to the order (usually by the product definition) and enforced mechanically. "It depends" names the parameter; the branching stays outside.

### Pattern: Strong Consistency Inside, Eventual Consistency Outside

Invariants are enforced atomically inside the order; everything it triggers beyond its boundary can be slow, fail or arrive late. Ordered steps with compensations, an outbox, idempotent integrations and tracing are the price of that boundary.

### Pitfalls

- **The document workflow** — statuses on a "request" record with a checkbox for who pays: the product, price, roles and availability are all there, unheard
- **The order as warehouse** — shipment, return, damage and stock events living inside the order until it *is* the inventory system
- **The order as brain** — one class that knows billing, warehouses, regulations and every exception; every new product is a change to it
- **Ordering first** — building the order before product, pricing, inventory and party exist; all of their rules leak into it

---

## Quality Checks

Before returning the model, verify:

- [ ] Complexity level is stated, and every skipped layer is marked *deliberately not modeled*
- [ ] Every line references a product type and a quantity; packages are single lines; no option is a flag
- [ ] Every state in the lifecycle is named, reachable and has a meaning; terminal states are marked
- [ ] Every specification key traces to the product definition; preferences are distinguishable from features; the order interprets no context key
- [ ] Every role has a scope and a cardinality; line-scope roles override per role; party data policy (reference / snapshot) is stated
- [ ] Every priced line records unit, total, source, definitive flag and value unit; the order total is absent while any line is unpriced; arithmetic checked by hand
- [ ] Every operation × state cell says allowed / refused, its external effect and its compensation; confirmation steps are ordered
- [ ] Every allocated line states strategy, shortage policy, where the reference is kept and what releases it
- [ ] Fulfillment status is separate from order status and mapped onto it; failure has a named reaction
- [ ] Payment model and orchestration style are chosen with a reason
- [ ] The order is downstream of every neighbour: commands and queries out, results and events in; no neighbour needs to know ordering
- [ ] Concept mapping is complete; Unmapped Concepts is present (even if empty)
- [ ] The worked walkthrough is answerable from the tables alone, includes one refusal and one routed question
- [ ] No (X) decision remains unconfirmed (Step 11.5); every clarifying answer or assumption is reflected in the model

---

## Example

**Input:** "Retail bank, mobile app. A customer taps *'I want a fast credit decision'* on a pending loan application. The decision can be for today or for tomorrow; today's costs a fee (PLN 40 net + 23 % VAT), tomorrow's is free. During a promotion, marketing pays the fee instead of the customer. Some customers pay the fee with loyalty points instead of money. The credit desk has limited daily capacity; when today's capacity is gone the customer may wait in a queue for a freed slot or switch to tomorrow. The customer may cancel until the decision is issued and gets the fee back. An analyst issues the decision, sometimes partly by hand; it may be issued today but be effective from tomorrow. Compliance must be able to see what was requested, by whom, at what price, who paid, and what actually happened."

**Detected level:** 8 — external processes (layers 1, 3–8 signalled; layer 2 trivially one line; layer 9 not needed).

**Output:**

```markdown
# Ordering Archetype Model: Fast Credit Decision

## Complexity Level
Level 8 — external processes. Layers modeled: identified intent, semantic scope, roles, recorded
price, change after placement, resource allocation, external processes. Deliberately not modeled:
composable scope (one line by construction) and many process families (one process; the loan
itself is a separate order process).

## Concept Mapping

| Domain Concept | Ordering Archetype | Notes |
|----------------|--------------------|-------|
| "I want a fast credit decision" | Order (one line) | The intent; not a document |
| Fast credit decision — today / tomorrow | Product type + specification key `mode` | Two variants of one product, not two products |
| Pending loan application | Line specification key `application` | The subject the decision concerns; owned by the lending module |
| Fee PLN 40 + VAT / free / 500 points | Line Pricing — calculated, definitive | Value unit differs: PLN or PTS; pricing decides |
| Marketing pays during promotion | Role payer held by Marketing | Same price, different payer |
| Customer, credit desk | Roles orderer/receiver, executor | The analyst is the executor's internal resource |
| Daily capacity of the credit desk | Allocation — order-driven; shortage policy waitlist-or-reject (customer's choice) | Capacity itself is inventory |
| Queue for a freed slot | Order state pending allocation | Order waits; inventory retries |
| Cancel until issued, fee back | Operation × state rule + refund compensation | Refund goes through payment / loyalty ledger |
| Decision issued (partly by hand) | Fulfillment status reported by the credit desk | Order does not know how |

## Unmapped Concepts
- Credit scoring and the decision content — lending module (fulfillment mechanics)
- Loyalty-point balances — accounting archetype (the order only initiates a transfer)
- Promotion eligibility rules — pricing / rules module; the order carries `campaign` in context

## Clarifying Questions & Answers
- Confirmation contract? → (A) allocate capacity, take payment or book the charge, start fulfillment — in that order
- Definitive price required at confirmation? → (A) yes; no estimates in this process
- Changes after confirmation? → (A) none; switching today↔tomorrow = cancel and place anew
- Cancellation? → (A) allowed in confirmed, pending allocation and processing; refused once issued
- Party data? → (A) reference only — every party is a registered bank customer or unit
- Allocation timing? → (A) order-driven, at confirmation; tomorrow's decision needs no slot
- Shortage policy? → (A) waitlist if the customer opts in, else reject with the tomorrow alternative
- Payment model? → (A) pay at confirmation (money or points); billing record when marketing pays
- Fulfillment failure? → (X — confirm) put to the business in Step 11.5 and left open; the model assumes cancel + refund and delivers it flagged

## Order & Lines

Order: a fast-decision request tied to one loan application; identifier `FD-…`; version for optimistic locking.

| Line | Product type | Quantity | Package? | Notes |
|------|--------------|----------|----------|-------|
| 1 | Fast credit decision | 1 | no | Variant chosen in specification |

## Lifecycle

| State | Meaning | Terminal? | Entered by |
|-------|---------|-----------|------------|
| draft | request composed in the app, priced | no | placement |
| confirmed | slot allocated, fee paid or charged, desk notified | no | confirm |
| pending allocation | today's capacity gone; waiting for a freed slot | no | confirm with waitlist policy |
| processing | analyst working on the decision | no | desk reports "started" |
| fulfilled | decision issued; effective-from recorded | no | desk reports "issued" |
| closed | fee settled, archived | yes | settlement |
| cancelled | withdrawn or failed; compensations done | yes | cancel / failure |

## Line Specification & Business Context

| Line | Specification shape | Keys (from product definition) | Execution preferences |
|------|---------------------|--------------------------------|------------------------|
| 1 | created on demand (the decision does not exist beforehand) | `mode` ∈ {today, tomorrow} (mandatory), `application` (mandatory), `payment_method` ∈ {money, points} (optional, default money) | `_contact_channel` (push / sms) — underscore prefix is the convention chosen here |

Business context: `channel = MOBILE`, `customer_segment`, `campaign` (promotion code), `order_time`. Interpreted by product (applicability) and pricing (fee, promotion, points tariff); never by the order.

## Parties & Roles

Party data: reference only; unregistered parties: no.

| Role | Scope | Cardinality | Source of assignment | Requirements |
|------|-------|-------------|----------------------|--------------|
| orderer | order | exactly 1 (A) | command (the logged-in customer) | — |
| receiver | order | exactly 1 (A) | same as orderer | — |
| payer | order | exactly 1 (A) | customer, or Marketing when `campaign` marks a promotion | — |
| executor | order | exactly 1 (A) | routed: the credit desk of the customer's region | desk must be enabled for fast decisions |
| payment receiver | order | exactly 1 (A) | fixed: the bank's settlement unit | — |
| line-scope roles | line | none allowed (R) | — | single line |

Instances: Customer C-1024 — orderer, receiver, payer (outside promotion); Marketing MK-01 — payer (during promotion); Credit Desk South — executor; Settlement Unit — payment receiver.

## Pricing

Sources allowed: calculated only; definitive price required at confirmation: yes.
Discount style: none — the promotion is expressed by the payer role, not by a discount (neither automatic nor explicit applies).

| Line | Source | Unit | Total | Value unit | Breakdown | Definitive? |
|------|--------|------|-------|------------|-----------|-------------|
| 1 (today, money) | calculated | 49.20 | 49.20 | PLN | base 40.00; VAT 23 % 9.20 | yes |
| 1 (tomorrow) | calculated | 0.00 | 0.00 | PLN | base 0.00 | yes |
| 1 (today, points) | calculated | 500 | 500 | PTS | fee 500 | yes |

Order total: equals the single line's total; none until the line is priced. Quantity is always 1, so unit = total.

## Operations by State

| Operation | draft | confirmed | pending allocation | processing | fulfilled | closed / cancelled |
|-----------|-------|-----------|--------------------|------------|-----------|--------------------|
| change `mode` / `payment_method` | allowed; re-price | refused — cancel and place anew (A) | refused (A) | refused | refused | no |
| set manual price | refused (A) | refused | refused | refused | refused | no |
| confirm | steps 1–3 below | no | no | no | no | no |
| cancel | set status | release slot, refund 49.20 PLN / 500 PTS or void the marketing charge, notify (A) | leave the queue; nothing to refund — payment has not run yet (A) | stop the analyst's work, refund or void (A) | refused — decision issued (A) | no |
| fulfillment report | no | accept | no | accept | accept ("settled" → closed) | no |

Confirmation steps in order: (1) allocate a desk slot for `mode = today` — on waitlist result go to pending allocation and skip steps 2–3 until allocated; (2) capture 49.20 PLN, or transfer 500 PTS via the loyalty ledger, or book a billing record for Marketing — on failure release the slot, stay in draft; (3) hand over to the credit desk — on failure refund or void, release the slot, stay in draft.
History mechanism: change records on the order (placement, price, payer, every state change with reason and actor).

## Allocation

| Line | Strategy | Shortage policy | Reference kept | Released on cancel by |
|------|----------|-----------------|----------------|-----------------------|
| 1, `mode = today` | order-driven at confirmation | waitlist if customer opted in, else reject with alternative "tomorrow" | slot id from inventory (in the specification after allocation) | cancel, via inventory |
| 1, `mode = tomorrow` | none | — | — | — |

## Fulfillment & Payment

Fulfillment status vocabulary: not started, started, issued, failed, settled → order status: started → processing; issued → fulfilled (with `effective_from` recorded); failed → cancelled with refund (X — confirm); settled → closed.
Aggregation across lines: single line, none.
Payment model: pay at confirmation for the customer (money capture or points transfer); billing record to Marketing's cost centre during promotions.
Orchestration style: direct — one dominant process, one executor; the desk reports back with commands.

## Boundaries & Integration

| Neighbour | The order asks it for | The order keeps |
|-----------|-----------------------|-----------------|
| Product & catalog | variants of the fast decision, mandatory keys, applicability for `campaign` | product type id, specification |
| Pricing | fee for {mode, payment_method, campaign, segment} | 49.20 PLN / 0.00 PLN / 500 PTS with breakdown |
| Party | customer, marketing unit, desk routing input | party ids, roles |
| Inventory | desk capacity slot for today | slot id, allocation result |
| Accounting / payment | money capture, points transfer, marketing charge, refunds | payment / transfer / billing references |
| Fulfillment (credit desk) | issue the decision | fulfillment status, decision id, effective-from |

## Worked Walkthrough
1. Monday 09:00, customer C-1024 taps the button, mode today, money. Order FD-1 in draft; pricing returns 49.20 PLN (40.00 + 9.20); payer = C-1024 (no campaign). Deciding concepts: line specification, line pricing, role policy.
2. Confirm. Step 1: inventory allocates slot S-17 → recorded in the specification. Step 2: 49.20 PLN captured, reference P-88. Step 3: Credit Desk South notified. State confirmed.
3. 09:40 the desk reports "started" → processing. 14:30 it reports "issued", decision D-501, effective from Tuesday → fulfilled; `effective_from = Tuesday` recorded.
4. Deviation A: on Wednesday another customer confirms at 15:50 when today's capacity is gone; opted into the queue → allocation result "waitlisted", order goes to pending allocation, no payment taken yet. At 16:10 a slot frees; inventory retries → allocated → steps 2–3 run → confirmed.
5. Deviation B: the queued customer cancels at 16:00 instead → allowed in pending allocation; leaves the queue; nothing to refund; cancelled.
6. Refused: the customer tries to switch a confirmed today-request to tomorrow → refused by the operation × state table; the app offers cancel and place anew (refund 49.20 PLN).
7. Routed: "how many decisions can the desk still take today?" → inventory, not the order; "how many points does C-1024 have?" → accounting.

## Implementation Notes
- Promotion is expressed by the payer role, not by a discount line; the price stays 49.20 PLN and Marketing's cost centre is charged — compliance sees who paid
- Points are a value unit returned by pricing; the order records "500 PTS" and initiates a ledger transfer; it knows nothing about point balances
- Fulfillment failure → cancel with refund is an assumption (X — confirm): it was put to the business in Step 11.5 and left open, so it is delivered flagged rather than silently; alternative: keep processing and route to a human
- Analyst identity and manual steps stay inside the credit desk; the order records only status, decision id and effective-from
- One line by construction; role policy forbids line-scope roles
- Level 9 not modeled: the loan application itself is a separate order process with its own lifecycle
```

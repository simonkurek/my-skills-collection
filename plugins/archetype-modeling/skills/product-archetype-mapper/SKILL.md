---
name: product-archetype-mapper
description: Transform domain requirements into a Product / Catalog Archetype model. Identifies complexity level (0–7), separates product definitions from instances, designs tracking strategies, feature types with constraints, catalog entries with validity, directed relationships, applicability conditions and packages with selection rules. Produces an implementable model with explicit concept mapping, unmapped concepts and integration boundaries.
argument-hint: "[domain requirements or feature description]"
---

# Product Archetype Mapper

Transform any domain where **something is offered, performed or settled according to a definition** into a product model. The "product" does not need to be a thing on a shelf — it can be a service, a tariff, a plan, an instalment, a fee, a medical procedure, a settlement, or a guarantee the system picks on the customer's behalf.

> *"Product is only the name of the archetype, and we in IT are bad at naming things."* A product is **anything that changes the state of our resources** — money, time, goods, people, space — and has rules, parameters and operational or financial consequences. The direction of money flow does not matter: discounts, returns and costs are products too.

**Output goal**: A complete, implementable model in which a new offer is **data, not code**: definitions separated from occurrences, every instance identifiable the way its business needs, features validated by the model, offers managed by marketing without touching definitions, relationships and applicability rules in one place, and packages that can be read as a specification of the offer.

## When to Use

**Use this skill when:**
- The same offering exists in several sizes, options or configurations, and those options drive price, operations or resource planning
- Business people say *package, plan, subscription, tariff, add-on, fee, instalment, settlement, bonus, limit* — or say "we don't sell anything" while every operation is a repeatable, definable event with parameters
- One thing can only be bought with another, is excluded by another, or upgrades to another; items must be recalled by batch or told apart by serial number

**Output is useful for:**
- Domain modeling sessions before implementation
- Deciding the boundary between the product module and pricing, inventory, ordering and billing

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the product archetype does not fit, and briefly explain why.

### The core question

> *"Is there something the organization offers, performs or settles that has a **definition** separate from its **occurrences** — with parameters, rules, and operational or financial consequences?"*

If **yes** → product archetype fits; go on to find how much of it is needed.
If the natural question is one of these, another archetype owns the concern:
- *"How much X does S have?"* → accounting (a balance with history)
- *"For how much, on what terms?"* → pricing (*"product is semantics, pricing is economics"*)
- *"Who is this, in which role, related to whom?"* → party
- *"How many are in stock, where, reserved for whom?"* → inventory (downstream of the catalog)
- *"What state is this delivery / visit / order in?"* → fulfillment or a state machine, not the product

Note the course's own position: nearly every organization has products, whether or not it uses the word. A ❌ below means *this concern is owned by a neighbouring archetype*, not *this business has no products*. If products exist but the domain needs only the shallow end of the ladder, that is level 0 or 1 — not a no-fit.

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "offered in several sizes / variants / options; options change cost, handling, resources" | ✅ Yes |
| "package / plan / subscription / tariff / add-on / settlement / instalment / fee" | ✅ Yes — mask words for products |
| "X cannot be bought without Y", "X excludes Y", "X upgrades to Y" | ✅ Yes — relationships or package rules |
| "one offer, thousands of copies each with its own customer and balance" | ✅ Yes — definition/instance confusion to fix |
| "recall this batch", "each unit has a serial number", "which dispatch run was it in?" | ✅ Yes — tracking strategy |
| "available only in Poland / only in the app / only to business customers / only in winter" | ✅ Yes — applicability |
| "the system, not the customer, picks the SLA / fee / surcharge" | ✅ Yes — still a product |
| "customer balance / quota / points earned and spent" | ❌ No — accounting |
| "price depends on weight, time, segment; VAT breakdown" | ❌ No — pricing (product tells pricing *what*) |
| "stock levels, locations, reservations" | ❌ No — inventory |
| "sent → in transit → delivered", "scheduled → done" | ❌ No — fulfillment lifecycle |
| "a single one-of-a-kind item with its own history" | ⚠️ Borderline — the definition *is* the specimen; catalog adds nothing |
| "one service, unchanged for 20 years, no variants planned" | ⚠️ Borderline — level 0; keep only the definition/instance split |

### Borderline cases — how to decide

- **Services**: the same thing as products. Every service is measurable (hours, nights, minutes); a service's lifecycle belongs to fulfillment, not to the product. *"Until you have a hard reason to separate services and products, don't."*
- **Unique items** (a collector's guitar, a painting): a special case of individual tracking with a unit of exactly one piece. Model the identity, skip the catalog.
- **Something the customer does not choose** (a system-selected SLA), and **things that cost us** (dunning letters, court fees, returns): still products — they have definitions, parameters and consequences. Customer choice and money direction are not the criteria.

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The product archetype requires a definable offering with occurrences, parameters, rules and
operational or financial consequences. This domain is a [ledger / valuation / party network /
inventory / workflow / ...] because:

- [specific reason from the requirements]
- The natural question is "[how much / for how much / who / where / what state]",
  not "what do we offer, in which variants, under which rules?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — what does the organization offer, perform or settle; in which variants; under which rules; and what has to happen downstream when a customer picks an option?"

---

### Step 1: Assess Complexity Level

The source course describes **six levels of product chaos** that companies pass through before discovering that the problem is not bugs but the absence of a product model. This skill adds level 0 (the course's explicit permission to model literally at small scale) and lifts the definition/offer split out as level 4, because the course teaches it as its own layer rather than as a stage of chaos. Level 1 is the foundation; the layers above it are independent — a domain may need packages without relationships, or applicability without a separate catalog. Identify **every layer that has a signal**, model each of them **and no further**, and report the level as the highest one modeled.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 0 | **Single stable offering** | One service or good, no variants expected, small system | (none — literal modeling is fine; keep definition and instance apart) |
| 1 | **Definition vs instance** | Several offerings; "a new product should be data, not code"; copies of the offer per customer | a class per offering; the offer and the contract in one record |
| 2 | **Identity & measure** | Serial numbers, batches, recalls, warranty; goods sold by piece here and by kilogram there | free-text units; no batch concept; "which 3.5% rate is current?" |
| 3 | **Features & metadata** | Colour, size, term, amount, weight with allowed ranges; mandatory vs optional; variants | a column per feature; validation in controllers |
| 4 | **Offer vs definition** | Campaign names, sales copy, categories, availability windows, channels, badges | marketing edits the definition; one definition renamed per campaign |
| 5 | **Relationships** | Upgrade paths, substitutes, complements, compatibility, exclusions | links in Excel and `if`s; foreign keys piling up on the "central" entity |
| 6 | **Applicability** | "Only in PL", "only in the app", "only over 18", "only business customers", "only in winter" | `if`s in every channel; app and backend disagree |
| 7 | **Packages** | Bundles with choice groups, "exactly one of", "up to two of", "if A then not B"; packages of packages | extra fields glued onto the key product |

**Guidance — which steps to run:**
- Level 0: the archetype is overkill. Run Steps 0–4 only, using the domain's own vocabulary (*massage type / massage*, not *product definition / instance*), state that no further layer is modeled, and stop. Skip Steps 5–11.5.
- Always: Steps 0–3, Step 4 (definitions and instances), Step 11 (boundaries), Step 11.5.
- Add one step per signalled layer: level 2 → Step 5; level 3 → Step 6; level 4 → Step 7; level 5 → Step 8; level 6 → Step 9; level 7 → Step 10. Packages are catalog citizens like any other definition, so a package layer often comes with a catalog layer — but neither implies the other; adopt each on its own signal.

Partial adoption is normal — *"archetypes are a way of modeling, not a ready recipe; from each archetype pick what fits your case."* Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps between the requirements and the archetype. Ask about **two categories** of questions in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Ask only about layers Step 1 selected, and only the questions whose answer changes the **shape** of the model. Take the simplest option for the rest and mark it *(assumed)*.

#### Category A — Standard product decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as design choices:

- **Vocabulary**: The archetype's generic terms (product definition / instance) or the domain's own (massage type / massage)? Generic once there are dozens of offerings; domain-specific for one stable offering.
- **Products and services**: One unified concept (default) or a light specialization for genuinely different needs (reservations, resource planning, SLA)?
- **Tracking per definition**: Individually (serial), by batch, both, neither (interchangeable) — and which offerings are one-of-a-kind?
- **Definition ↔ catalog entry**: Definition *is* the offer (no catalog) / 1:1 / several definitions under one entry when the customer experiences them as one thing / several entries per definition for campaigns?
- **Where applicability lives**: On the definition (intrinsic to what the product is) / on the catalog entry (comes from the sales context: channel, campaign, market) / both?
- **Relationships**: Directed only, mirror pairs stored explicitly, or a policy that creates the mirror for compatibility? Transactional, or eventual with a cleanup cadence (hourly / per minute / event-driven)?
- **Selection rules**: Cardinality only (exactly one / optional / at least one / between min and max) or also conditional (if A then B, if A then not B) and logical combinations?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the archetype supports but the requirements do not mention**. Ask only where the requirements hint that the dimension exists. Do not limit yourself to this list — reason freely:

- **Alternative units**: Is the same product measured differently per channel (kilograms wholesale, pieces retail)? Is conversion needed?
- **Batch structure**: What does a batch carry — quantity, production date, use-by date, a serial range? Which of these matter here?
- **Feature vs metadata**: Is a property potentially variable (feature) or definitional (metadata — changing it means a different product)? Which features need default values?
- **Dependent features**: Does choosing one feature restrict another ("matrix headlights only in Sport")? Split into separate definitions, accept the combinatorics, or point to a solver?
- **Instance creation moment**: Do instances exist before sale (goods at production or goods-receipt) or only at order (services, made-to-order)?
- **Relationship-defining rules**: No self-relationship? No cycles in upgrade paths? Substitutes only within a category? Maximum number of complements? Seasonal or certification-bound compatibility?
- **"Requires" / "recommends"**: Hard requirements ("consultation requires ECG") — a package rule, a relationship, or an applicability condition?
- **Identical items with tracking data**: Should supplying a serial or batch for an interchangeable product be rejected, or silently ignored?

Collect answers before proceeding. If the user cannot answer, document the assumption made in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

Always include **"To zależy / It depends"** as an explicit option in every `AskUserQuestion` call — do not rely on the automatic "Other" fallback. Place it as the last option in each question. If the user selects it, treat it as a **variable policy**:

- Name the *policy* the model will accept (relationship-defining policy, applicability constraint supplied per entry, selection rule set per package, tracking strategy per definition)
- Note in **Implementation Notes** that the concrete rules are supplied from outside (configuration, the product team's definitions office, tenant settings) and the model only *enforces* what it receives
- Do **not** model the decision logic inside the product model

This is the correct outcome — the archetype is a small rules engine over definitions; which rules apply is data.

#### If `AskUserQuestion` is not available

Do not stop. Use these defaults, list each in **Clarifying Questions & Answers** marked *(assumed)*, and treat every Category B gap as *not wanted* unless the requirements imply it:
- generic vocabulary; products and services unified
- tracking: identical unless the requirements name a serial, contract, tracking or certificate number (→ individual), a batch, run, recall or expiry (→ batch), or both
- 1:1 definition ↔ catalog entry; a campaign or promotion is an **additional** entry alongside the standing one
- a fixed requirement of the product (which device, which specialization, fasting needed) is **metadata**; a condition on the situation at order time (customer age, channel, equipment available now) is **applicability** on the definition, moved to the entry only when the rule names a channel, campaign or market
- directed relationships, mirrors stored explicitly; eventual consistency with existence checked at creation
- cardinality rules, plus if-then where the requirements state a conditional

---

### Step 3: Map Domain Concepts to Product Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Product Archetype | Notes |
|----------------------|-------------------|-------|
| [domain noun/verb]   | Product Definition / Product Instance / Tracking Strategy / Serial Number / Batch / Unit / Feature Type / Feature Value / Metadata / Catalog Entry / Validity / Relationship / Relationship Policy / Applicability Constraint / Context Dimension / Package Definition / Choice Group / Selection Rule / Package Instance | [why] |
```

Listen for mask words: *settlement, package, subscription, plan, limit, instalment, fee, bonus* are products. A per-industry disguise — a *procedure* in medicine, a *tariff* in telecom, a *repayment plan* in banking, an *order* in logistics — is usually a product definition with instances.

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear product archetype equivalent:
- [concept] — [reason it doesn't fit / decision needed / which archetype owns it]
```

This section must be present even if empty (`None identified`). Prices, balances, stock levels, delivery states, results produced by performing an instance, and approval workflows land here with the owning archetype named.

---

### Step 4: Separate Definitions from Instances

Find every **product definition** — the template, "what it is" — and its **instances** — the occurrences, "what was actually made, sold or performed".

**Detection signals:**
- A noun used both for the offer and for the customer's copy ("mortgage" = the bank's product *and* Kowalski's contract; "parcel" = the 24h service *and* this shipment) — split it
- "Every X has its own history / status / fate" → X is an instance; its template is the definition
- "The offer changed and old records know nothing about it" → definitions were copied into instances

**Product definition** — the minimal skeleton is an identifier, a name (customer-facing, never an identifier) and a description; *"we need nothing more to begin with."* Later layers add unit and tracking strategy (Step 5), feature types and metadata (Step 6), applicability (Step 9). A definition is stable and changes rarely. Some definitions never go on sale: internal technical products, archived products still being settled, prototypes.

**Product instance** — a reference to its definition, the identifiers its tracking strategy demands (Step 5), feature values (Step 6) and, for simple products, a quantity in the definition's unit. Anything about the instance's *state* — reserved, shipped, delivered, in progress — and anything *produced by performing it* — a test result, a report, a measurement — belongs to fulfillment, inventory or the domain's records, not here.

**When does an instance come into being?** Decide per definition:
- **Goods**: before any customer — at manufacture or goods-receipt: *"200 bottles of whisky from batch 2024-001. Those instances already exist before the customer walks into the shop. We only reserve or issue one of them."*
- **Services and made-to-order products**: at order time (*"we don't have five hundred massages ready to sell"*) — the instance is created because a real event will consume a resource

**Rule:** products and services share one concept. Every service is measurable (hours, nights, minutes, per piece). Add a specialization only for a hard reason (reservations, resource planning, SLA) and say why.

**Three roles read the same split differently** — producer: recipe vs manufactured unit; seller: catalog offer vs stock; customer: what they browse vs what they receive. Name the role whose perspective the requirements take.

---

### Step 5: Define Identity, Tracking and Measure

For each definition, decide **how instances are told apart**, if at all, and **in what unit** they are measured.

**Tracking strategies** — the stable part of a definition and the thing the model enforces when an instance is created:

| Strategy | What an instance must carry | Typical examples |
|----------|-----------------------------|------------------|
| One-of-a-kind | its own identity; unit is exactly one piece | a specific guitar, a painting |
| Individually tracked | a serial number (contract number, tracking number, VIN, IMEI) | phone, mortgage, parcel, policy, subscription period |
| Batch tracked | a batch reference | milk, medicines, a day's dispatch run |
| Individually **and** batch tracked | both | a TV (serial for warranty, batch for recalls) |
| Identical / interchangeable | neither — only a quantity | screws, rice, electricity, consulting hours |

**Rule:** the strategy lives on the definition, and instance creation **refuses** anything that violates it — no serial where one is required, no batch where one is required. This is the design decision: for identical items, decide whether supplied tracking data is *rejected* (strict, the reference's choice) or *ignored*.

**Convention:** one-of-a-kind is a flavour of individual tracking (the course's stance) and may or may not be a distinct strategy. Even identical units have non-identical **batches** — add batch tracking whenever recalls, quality control or "which run was it in" matter. A shared collection event the domain does not call a batch (one blood sample, one dispatch run) is a batch if traceability matters; otherwise it is a package (Step 10).

**Identifier family** — four different questions:
- **Product identifier** — *which kind of product*; GTIN for retail, ISBN for books, an internal id for anything that exists only in our domain. *"It doesn't matter as long as it is consistent."*
- **Serial number** — *which specimen*; assigned by the manufacturer or the business, unique only **within a definition**, format varies by industry (a VIN or IMEI carries its own validation)
- **Batch** — *from which run*; an example structure: the definition it belongs to, a quantity in the definition's unit, production date, use-by date, optional serial range, comments. *"Perhaps some of these don't occur in your business; the idea matters more."*
- **Instance id** — a technical key tying it together, because not every product has a serial and serials are not globally unique

**Unit of measure** — every product is measurable: pieces, litres, kilograms, hours, nights, months. Record a **preferred unit** per definition and, when the same product is sold by kilogram wholesale and by piece retail, the set of acceptable units and whether conversion is needed. Instance quantities must be in a unit the definition accepts. A package has no unit of its own beyond "number of sets" (Step 10).

---

### Step 6: Define Features, Metadata and Defaults

Decide what can **vary** per instance, what is **identity** of the definition, and what is merely a **convenient starting point**.

| | Meaning | Test | Examples |
|---|---------|------|----------|
| **Feature type** | potential variability; the customer or configuration may set it | *"will it certainly change between instances?"* | colour, size, deposit term, parcel weight, appointment date |
| **Metadata** | identity; changing it means a different definition | *"would this ever change without it being a different product?"* | currency of a EUR deposit, tariff G11 = single-zone households, required device, required specialization, fasting needed |
| **Default value** | UX convenience the customer may override | *"does the business want a suggested value?"* | 12-month term, e-invoice, USD 1,000 deductible |

**Rule:** *"Feature = potential variability. Metadata = identity."* A feature with a single allowed value is technically metadata; choose on readability and extensibility, and say which you chose. A **fixed requirement of the product** (which device, which staff qualification, fasting) is metadata; whether that requirement **is met right now** (device available, patient fasting today) is applicability (Step 9) — the course's medical example puts *apparatus, category, fasting* in metadata and *adult, department, equipment available* in applicability.

**Feature type** = a name + a **value constraint**. The constraint answers two questions: what kind of value (text, integer, decimal, date, boolean…) and is this value valid. Describe constraints by shape; the set is extensible:

| Shape | Example |
|-------|---------|
| one of an allowed set | colour ∈ {red, blue, green}; channel ∈ {mobile, web, branch} |
| integer range | term 1–60 months; year of production 2020–2024 |
| decimal range | amount 1 000–1 000 000 PLN; weight 0.1–125 kg |
| date range | pickup date within the next 30 days; expiry not after 2030 |
| text pattern | batch code `abc-2024-pl`; internal id `item 000456` |
| any value of a declared kind | a doctor's free-text note — *"the system trusts the human but still knows it is text"* |

Each feature type on a definition is **mandatory or optional** — *"this T-shirt has no size"* is an error; *"this T-shirt has no print"* is fine.

**Feature value** = feature type + value, validated against the constraint **at creation**. An instance is refused if a mandatory feature is missing or if it carries a feature the definition does not declare. *"No ifs in controllers — the domain already knows."* One value per feature type per instance; the name is the identity.

**Dependent features** ("card colour selectable only when term > 5 years"): prefer **splitting into separate definitions** with narrowed constraints (a short-term and a long-term agreement) — this is a design decision; the alternatives are accepting the combinatorial explosion or escalating to a constraint solver.

---

### Step 7: Separate the Offer from the Definition

If the domain has campaigns, channels, seasons or marketing names, model the **catalog entry** — *"whether and how we sell it"* — apart from the definition — *"what it is"*.

A catalog entry carries: its own id (not the product identifier — *"we are no longer speaking of something that exists but of something we show the customer"*), a **display name** (*Lokata Premium 7.5% Online* for a definition called *Deposit Account Type 2025*), sales copy, **categories** for navigation, a **validity** window, and free-form **campaign metadata** (badges, channels, priorities, landing pages). It **references** the definition and duplicates none of its structure or validation.

**Two offices:** the *definitions office* (product team) creates and changes definitions; the *sales office* (marketing) publishes and withdraws entries and needs rich search — by category, by availability date, by feature values, by metadata. *"The definition stays constant while the offer can change daily."*

**Cardinality** — a semantic decision, not a technical one:
- **1:1** (default): simplicity, unambiguous reporting; right for banking, telecom, e-commerce. A configurable product stays 1:1 whether the front end renders a configurator or many tiles.
- **Several definitions under one entry**: *"if from the customer's point of view it is one and the same experience"* — three 45-minute massage variants sold as *Relaxing massage 45 min*. If the differences are business, cost or operational, split.
- **Several entries per definition**: the same deposit in the Q1, Q2 and Q3 campaigns, each with its own window — the reference's choice.

Not every definition needs an entry. Components sold only inside a package, internal technical products and archived products still being settled have none — that is the normal case, not a gap.

**Validity** lives on the catalog entry, never on the definition: a product exists indefinitely as a concept while its saleability is finite. A window may be open at either end; **discontinuation closes the window** — nothing is deleted. Re-offering after a gap is a new entry.

**Convention:** validity is day-granular with inclusive bounds; overlapping windows for one definition are the caller's responsibility. Name the alternative if the domain needs time-of-day or overlap detection.

Versioning of definitions, approval and publication workflow, and change history are real product-management needs the archetype sources leave out — list them in Unmapped Concepts as a layer above the catalog if the requirements ask for them.

---

### Step 8: Define Relationships Between Products

Model links between **definitions** as a **closed set of directed relationship kinds**. Unlike party, where relationships are open and effectively infinite, *"the world of the offer must be consistent and controlled."*

| Kind | Meaning | Direction reads as |
|------|---------|--------------------|
| upgradable to | can be raised to a higher version | standard plan → premium plan |
| substituted by | offer this when the other is unavailable | laptop 2023 → laptop 2024 |
| replaced by | withdrawn, succeeded by | old card line → new card line |
| complemented by | completes a set; drives cross-sell | burger → fries (*the burger initiates*) |
| compatible with | works together technically | charger → phone |
| incompatible with | mutually exclusive | business plan → consumer plan |

**Rule:** relationships are **directed**; the direction carries meaning (*"if Standard is upgradable to Premium, the reverse does not exist"*). Compatibility is the natural symmetric case. This is a design decision: store one relationship and read it as mutual, store the mirror pair explicitly, or let a policy create the mirror. *"Two asymmetric relationships make a symmetric one; the reverse is impossible"* — asymmetric storage is more universal. Whatever you choose, be consistent.

**Rule:** relationships are **independent entities** with their own identity and store, not lists on the definition — at a thousand products and tens of thousands of links, lists load the whole network and lock entities on every change. Packages participate exactly like simple products.

**Consistency:** relationships are *soft business metadata* — recommendations, catalog information, soft exclusions — not transactional invariants. **Eventual consistency is sufficient**; a stale upgrade hint for an hour is a lost suggestion, nothing more. Pick the cleanup cadence to match tolerance. Design decision: check that both products exist when the relationship is *defined* (the reference does) and tolerate staleness afterwards, or go fully transactional.

**Relationship-defining policy:** creating a relationship is itself rule-governed. Put the rules in a policy consulted every time, not in `if`s in every entry point — *"we want the model to remember for the human."* The rules are the ones elicited in Step 2 (no self-relationship, no cycles, same category, maximum complements, seasonal or certified compatibility).

**Extending the set:** *"requires"* and *"recommends"* in requirements are usually a **package selection rule** (hard requirement: consultation requires ECG → a rule in the pathway package) or *complemented by* (soft). Add a kind only when neither fits, and do it consciously.

---

### Step 9: Define Applicability Conditions

*"Nie dla psa kiełbasa"* — not everything is for everyone. Applicability rules say **when a product makes sense in a context**; they are conditions of the situation, not properties of the product.

**Applicability context** — a bag of whatever parameters matter: country, channel, customer type, age, tier, date, season, contract status, volume, department, equipment or staff available. *"The model does not have to know them up front."* List the dimensions the domain uses; party supplies the ones about people.

**Applicability constraint** — a composable boolean condition over the context: equals, in a set, greater / less than, between, combined with **and / or / not**. A definition without a constraint is applicable everywhere. Worked shape: *country ∈ {PL, DE} AND age > 18 AND NOT channel = desktop* — a 25-year-old Polish resident qualifies, a 17-year-old German does not.

**Convention:** a comparator over a parameter the context does not supply is **not satisfied** (fail closed). This applies to the *leaf*: a missing parameter under a **not** makes the wrapped comparator false and the negation true. Where a negated rule must also be conservative, state the requirement positively (*channel ∈ {mobile, web}* rather than *NOT channel = desktop*). Name the alternative — a distinct "unknown, ask" outcome — if the domain needs it.

**Placement — a design decision with a test:**
- On the **definition** when the condition is part of what the product is (a paediatric service for patients under 16, a mobile-only subscription)
- On the **catalog entry** when the condition comes from the sales context — one travel insurance definition offered through an *Online* catalog (adults, individuals), an *Agent* catalog (licensed companies) and a *Promo Winter* catalog (December–January). The definition stays stable; the offer becomes configurable.
The reference keeps applicability on definitions only; the course recommends both. Say which you use and why.

**Rule:** applicability must be **part of the model, not `if`s in service code or filters in SQL** — as long as every channel polices it its own way, consistency is an illusion. The same mechanism may gate relationships. This is a small boolean-logic engine; if rules become a configurator (Tesla, insurance packages), the natural extension is a SAT/SMT solver above it, not more `if`s.

---

### Step 10: Design Packages

A package is *"not a marketing shortcut — a strategic business entity: the way an organization builds value by combining products."* A package is **not a list of products** but **choice groups plus selection rules**: *"this package must contain something from this list of insurance products"*, not *"this package contains Insurance A"*.

**Package definition** — shares with a simple definition: identifier, name, description, metadata, categories, applicability, tracking strategy (a physical bundle or a service with its own lifecycle needs one). Does **not** have: a unit (only "number of sets" — avocados by piece and potatoes by kilogram cannot share a measure) or its own feature types (they are the sum of the components'). This is why the archetype puts simple and package definitions **side by side under one product abstraction** (the Composite pattern) — not an optional structure on the simple definition (dead fields on every simple product, every operation first asking whether the structure is even there), not inheritance (a package is *a different kind* of product, not a more detailed one).

**Choice group** — a named set of definitions that are *equivalent options in this context*: "Insurance" = {standard, extended, partner}. A group with a single option is normal — it is how a mandatory or optional **component** is expressed. A group may contain other package definitions — packages of packages (*Office Setup* = Hardware Package + Software Package).

**Selection rules** — how one may pick from the groups; describe by shape:

| Shape | Reads as | Example |
|-------|----------|---------|
| between min and max from a group | exactly one (1..1), optional (0..1), up to two (0..2), at least one (1..∞) | one tariff plan; up to two accessories |
| if A then B | conditional requirement | if business account then exactly one business card |
| if A then not B | conditional exclusion | if international transport then not standard insurance |
| and / or / not over rules | logical combination | (basic card OR premium card) AND (if premium then extended insurance AND NOT basic insurance) |

All rules attached to a package must hold. **Convention:** cardinality counts **quantities**, not distinct products (one product with quantity 2 counts as 2); name the alternative if the domain counts distinct choices. *"In many businesses the cardinality rule alone is sufficient."*

**Package instance** — the package's own identity per its tracking strategy plus the **selected instances** (instance + quantity). Validation reduces each selected instance to its definition and checks the package's rules — *"rules are not interested in serial numbers or batches; they look only at definitions."* An instance whose content violates the rules **cannot be created**.

**Rule:** a package's rules check its **immediate** selection. "Is a valid Tracking Package present?" is the outer package's question; "is the Tracking Package's own content valid?" is answered when that nested instance is built. A rule spanning two levels (phone in the bundle, plan inside a nested pack) must be placed where both are directly visible.

Everything that applies to products applies to packages: catalog entries, search, metadata, applicability, relationships (*basic cardiac package → upgradable to → extended cardiac package*).

---

### Step 11: Define Boundaries and Integration

Product is a **generic business capability** — *"just as accounting manages value, product manages what can be sold, offered, settled or performed at all."* It is the **upstream** for the money-making processes and should be a **module or service with its own API, store and team**, not a shared library (binary coupling, blurred ownership, no persistence for a stateful model, contextual drift).

| Neighbour | Boundary | Direction |
|-----------|----------|-----------|
| **Party** | supplies context and authorization: who buys, who may perform, which tier — used in applicability | product depends on party |
| **Pricing** | *"product says what; the price list says for how much and on what terms"*; no price attributes on definitions | separate; pricing consumes definitions |
| **Billing / Accounting** | product tells billing *what* to settle; billing tells accounting *how much and how to post* | downstream |
| **Ordering** | needs the catalog to know what can be sold and configured | downstream |
| **Inventory** | tracks the physical side of instances — availability, quantity, location, reservations; creates no definitions | downstream |
| **Fulfillment** | owns realization lifecycles and their outcomes (sent → delivered; scheduled → done; the result of a test); *"depends on the catalog but is not part of it"* | downstream |

**Rule:** the product module *"does not know why anyone will use the product. It only knows what the product is, what features, relationships, constraints and availability it has."* It may depend on party and pricing but must not leak their logic.

Optional analytical layer: the definitions and relationships form a **graph** — cycle detection in upgrade paths, resource conflicts between procedures, ordering of steps in a pathway. Mention it; do not model it here.

---

### Step 11.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:

- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred from a structural rule of the archetype (state the rule)
- **(X)** — neither: assumed silently

Walk every Category A and Category B question from Step 2, every block marked **Rule**, **Convention** or **design decision** in Steps 4–10, and every row of the definitions, feature, catalog, relationship, applicability and package tables. Typical hiding places for (X): the instance creation moment, feature-vs-metadata calls, catalog cardinality, applicability placement, relationship consistency, quantity-vs-distinct counting, what was pushed to a neighbour.

**For every (X) decision found:**

1. If the decision has low impact (purely technical, easily changed): mark as explicit assumption in Implementation Notes.
2. If the decision affects business behaviour (cardinality of catalog entries, applicability placement, relationship consistency, what a package may contain): **stop and ask** using `AskUserQuestion` before delivering the model — or, if it is unavailable, take the Step 2 default and mark it *(assumed)*.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

```markdown
# Product Archetype Model: [Domain Name]

## Complexity Level
[Highest level modeled, layers with a signal, layers deliberately not modeled]

## Concept Mapping

| Domain Concept | Product Archetype | Notes |
|----------------|-------------------|-------|
| ...            | ...               | ...   |

## Unmapped Concepts
[List with owning archetype, or "None identified"]

## Clarifying Questions & Answers
[Question → answer, marked (asked) / (assumed) / (it depends → policy)]

## Product Definitions & Instances

| Definition | Kind | Unit | Tracking | Instance created | Notes |
|------------|------|------|----------|------------------|-------|
| [name] | simple / package | [unit] | [strategy] | before sale / at order | [what the instance represents] |

(Unit and Tracking columns only if the identity/measure layer is modeled — otherwise drop them)

## Identity & Tracking   (omit if no identity/measure signal — say so)
[Identifier scheme, serial number formats, batch structure, unit sets]

## Features & Metadata   (omit if no feature signal — say so)

| Definition | Feature type | Constraint | Mandatory | Default | Notes |
|------------|--------------|------------|-----------|---------|-------|

| Definition | Metadata | Value | Why identity, not feature |
|------------|----------|-------|---------------------------|

## Catalog Entries & Validity   (omit if no offer-vs-definition signal — say so)

| Entry | Display name | References | Categories | Valid from | Valid to | Entry-level applicability / metadata |
|-------|--------------|------------|------------|-----------|---------|--------------------------------------|

Cardinality decision: [1:1 / several definitions per entry / several entries per definition]

## Relationships   (omit if no relationship signal — say so)

| From | Kind | To | Mirror? | Notes |
|------|------|----|---------|-------|

Storage: [independent entity]; consistency: [...]; defining policies: [...]

## Applicability Conditions   (omit if no applicability signal — say so)

| Product / entry | Placed on | Condition | Context dimensions |
|-----------------|-----------|-----------|--------------------|

Missing-parameter behaviour: [...]

## Packages   (omit if no package signal — say so)

### [package name]
Tracking: [...]  Unit: sets

| Choice group | Options | Rule |
|--------------|---------|------|

Cross-group rules: [if A then (not) B ...]

Worked validations:
| Selection | Outcome | Rule that decided |
|-----------|---------|-------------------|

## Boundaries & Integration
[What the product module owns; what each neighbour owns; upstream/downstream]

## Implementation Notes
[Key decisions, assumptions for unanswered questions, layers deliberately not modeled, conventions chosen and their alternatives]
```

---

## Common Patterns & Pitfalls

### Pattern: A New Product Is Data, Not Code

The whole archetype exists so that adding an offering, a variant, a rule or a bundle is a **catalog edit, not a deployment**. When a step produces something that would need a new class per offering, a new column per feature, or a new `if` per rule, the model has slipped a level down the chaos ladder. *"The system doesn't support the offer — it simulates it."*

### Pattern: Three Worlds, One Product

Definition (*what it is* — the product team), catalog entry (*whether and how we sell it* — marketing), instance (*what actually exists* — operations and inventory). The border between them is always there; *"the only question is how thick the line is on your Kremenaros."* Not every domain needs all three, but no domain may collapse definition into instance.

### Pattern: Listen for Products Where Nobody Sees Them

*"Don't look for a Product table. Listen to how people in the company talk."* Instalments, interest, dunning letters and court costs are products; a settlement is a package. Tests, consultations and results are products; a diagnostic pathway is a package. A system-selected SLA is a product. If it has rules, parameters and consequences, model it.

### Pitfall: The Central Entity That Eats the Domain

A credit card number added to the account, then a loan id, an insurance id, a wallet alias — *"the class representing an account becomes the centre of the universe"* and hundreds of places check whether a field is null. Those are relationships (Step 8) or a package (Step 10).

---

## Quality Checks

Before returning the model, verify:

- [ ] Complexity level is stated, with layers modeled and layers deliberately not modeled
- [ ] Every definition is separated from its instances, and the instance creation moment is stated
- [ ] Products and services share one concept unless a hard reason is given
- [ ] No price, balance, stock level, delivery state or performance result appears on a definition or instance
- [ ] *If identity/measure is modeled*: every definition has a unit and a tracking strategy; every strategy names what an instance must carry
- [ ] *If features are modeled*: every property is classified as feature, metadata or default; features are mandatory/optional and have a value constraint by shape
- [ ] *If the catalog is modeled*: entries reference definitions and carry validity; the cardinality decision is stated; discontinuation never deletes
- [ ] *If relationships are modeled*: directed, typed from the closed set (or an extension is justified), stored independently, with a consistency mode and defining policies
- [ ] *If applicability is modeled*: rules name their context dimensions and their placement (definition vs entry)
- [ ] *If packages are modeled*: choice groups + rules, no unit or own features, every worked validation names the rule that decided it, nested content validated on the inner instance
- [ ] Concept mapping table is present and complete; Unmapped Concepts section is present (even if empty) with owning archetypes named
- [ ] No (X) decision remains unconfirmed (Step 11.5); every clarifying answer or assumption is reflected in the model
- [ ] Boundaries name party, pricing, inventory, ordering, billing and fulfillment

---

## Example

**Input:** "Courier company. We sell domestic and international express transport. Parcel weight 0.1–125 kg, size S/M/L/XL. Every shipment must be insured — standard, extended or a partner's policy — but standard insurance is not allowed for international transport. Optional add-ons: cold chain, fragile handling, pickup from the customer; at most two per shipment. Pickup is only for business customers ordering at least 5 shipments. Tracking is mandatory: system statuses or active GPS monitoring. Notifications (SMS, e-mail, webhook) are optional. Every shipment gets a tracking number and belongs to a daily dispatch run, so that if a run fails we know which shipments are at risk. Marketing sells the whole thing as 'Transport Premium' and wants a winter promotion in December and January sold online only. Extended insurance should be suggested as an upgrade of standard."

**Detected level:** 7 — packages with choice groups and a conditional rule. Layers with a signal: 1, 2, 3, 4, 5, 6, 7.

**Output:**

```markdown
# Product Archetype Model: Courier Transport Premium

## Complexity Level
Level 7. Modeled: definitions/instances (1), tracking & units (2), features (3), catalog with a
promo window (4), one upgrade relationship (5), applicability on pickup and on the promo entry (6),
a package of packages (7). Not modeled: product versioning and approval workflow (no signal).

## Concept Mapping

| Domain Concept | Product Archetype | Notes |
|----------------|-------------------|-------|
| Domestic / international express | Product Definition (simple), individually + batch tracked | the service being sold |
| A shipment | Product Instance of a transport definition | created at order; tracking number + dispatch run |
| Tracking number | Serial Number | unique within the transport definitions |
| Daily dispatch run | Batch | "which shipments are at risk" |
| Weight, size | Feature Types on transport definitions | mandatory; decimal range / allowed set |
| Insurance, add-ons, tracking and notification options | Product Definitions (simple) | eleven component definitions |
| "must be insured, one of three" | Choice Group + exactly-one Selection Rule | |
| "standard not allowed for international" | Conditional Selection Rule (if A then not B) | cross-group |
| "at most two add-ons" | Selection Rule 0..2 | |
| "pickup only for business customers ≥ 5 shipments" | Applicability Constraint on the pickup definition | intrinsic to the product |
| Transport Premium | Package Definition (package of packages) | |
| "Marketing sells the whole thing as Transport Premium" | Catalog Entry (standing, open window) | display name for the package |
| Winter promo, online only | Catalog Entry with validity + entry-level applicability | sales context |
| "extended suggested as upgrade of standard" | Relationship: standard → upgradable to → extended | soft, directed |

## Unmapped Concepts
- Surcharge for add-ons and international transport — pricing archetype (product tells pricing *what*)
- Shipment status (sent / in transit / delivered) — fulfillment; not part of the product model
- Vehicle and space planning driven by size and weight — downstream consumer of feature values

## Clarifying Questions & Answers
- Vocabulary → generic (asked); products and services unified (assumed)
- Tracking of shipments → individually and by batch (R); shipments without either are refused (I — Step 5 rule); insurance → individually (assumed)
- Definition ↔ entry → several entries per definition: one standing entry plus one winter-promo entry (R)
- Applicability placement → pickup rule on the definition, promo channel rule on the entry (asked)
- Relationship direction → directed, no mirror (assumed); consistency → eventual, existence checked at creation, hourly cleanup (assumed)
- Selection rules → cardinality + one if-then-not (R)

## Product Definitions & Instances

| Definition | Kind | Unit | Tracking | Instance created | Notes |
|------------|------|------|----------|------------------|-------|
| Domestic Express | simple | shipment | individual + batch | at order | one shipment = one instance |
| International Express | simple | shipment | individual + batch | at order | |
| Standard / Extended / Partner Cargo Insurance | simple | certificate | individual | at order | three definitions; partner-issued, our identifier |
| Cold Chain / Fragile Handling / Pickup Service | simple | piece | identical | at order | three add-on definitions |
| System Tracking / Active Monitoring | simple | piece | identical | at order | two definitions; monitoring = GPS + temperature |
| SMS / E-mail / Webhook notification | simple | piece | identical | at order | three definitions |
| Tracking Package | package | sets | identical | at order | nested |
| Notification Package | package | sets | identical | at order | nested |
| Transport Premium | package | sets | individual | at order | order number as serial |

## Identity & Tracking
- Product identifier: internal ids (services exist only in our domain)
- Serial numbers: tracking number on shipments (text, internal format); certificate number on insurance;
  order number on the Transport Premium instance
- Batch: dispatch run — date, quantity of shipments (unit: shipment), comments. Shipments without a
  tracking number or without a dispatch run are refused at creation.

## Features & Metadata

| Definition | Feature type | Constraint | Mandatory | Default | Notes |
|------------|--------------|------------|-----------|---------|-------|
| Domestic / International Express | weight_kg | decimal range 0.1–125 | yes | — | 130 kg and 0.05 kg are refused |
| Domestic / International Express | size | one of {S, M, L, XL} | yes | M *(assumed)* | |

| Definition | Metadata | Value | Why identity, not feature |
|------------|----------|-------|---------------------------|
| Domestic Express | scope | domestic | a different scope is a different product |
| International Express | scope | international | |
| Active Monitoring | sensors | gps, temperature | definitional |

## Catalog Entries & Validity

| Entry | Display name | References | Categories | Valid from | Valid to | Entry-level applicability / metadata |
|-------|--------------|------------|------------|-----------|---------|--------------------------------------|
| TP-STD | Transport Premium | Transport Premium (package) | transport, premium | open | open | — |
| TP-WINTER | Transport Premium — Winter | Transport Premium (package) | transport, premium, promo | 2026-12-01 | 2027-01-31 | channel = online; badge "winter" |

Cardinality decision: several entries per definition — one definition (Transport Premium) carries two
entries with different windows and channels; each entry references exactly one definition. Withdrawing
the promo early closes TP-WINTER's window; nothing is deleted. The thirteen component definitions have
no entries of their own: they are sold only inside the package.

## Relationships

| From | Kind | To | Mirror? | Notes |
|------|------|----|---------|-------|
| Standard Cargo Insurance | upgradable to | Extended Cargo Insurance | no | up-sell hint |

Storage: independent entity; consistency: eventual with hourly cleanup, both products checked to exist
at creation; defining policies: no self-relationship, no cycles in upgrade paths.

## Applicability Conditions

| Product / entry | Placed on | Condition | Context dimensions |
|-----------------|-----------|-----------|--------------------|
| Pickup Service | definition | customer_type = business AND shipments_in_order ≥ 5 | customer_type, shipments_in_order |
| TP-WINTER | catalog entry | channel = online | channel |

Missing-parameter behaviour: not satisfied (fail closed). A context without customer_type cannot
select Pickup Service.

## Packages

### Tracking Package
Tracking: identical  Unit: sets

| Choice group | Options | Rule |
|--------------|---------|------|
| Tracking | System Tracking, Active Monitoring | exactly one (1..1) |

Worked validations:
| Selection | Outcome | Rule that decided |
|-----------|---------|-------------------|
| System Tracking | valid | 1..1 satisfied |
| empty | rejected | 1..1 requires 1 |

### Notification Package
Tracking: identical  Unit: sets

| Choice group | Options | Rule |
|--------------|---------|------|
| Notification | SMS, E-mail, Webhook | optional (0..1) |

Worked validations:
| Selection | Outcome | Rule that decided |
|-----------|---------|-------------------|
| Webhook | valid | 0..1 satisfied |
| empty | valid | 0..1 allows 0 |

### Transport Premium
Tracking: individual (order number)  Unit: sets

| Choice group | Options | Rule |
|--------------|---------|------|
| Transport | Domestic Express, International Express | exactly one (1..1) |
| Insurance | Standard, Extended, Partner | exactly one (1..1) |
| Add-ons | Cold Chain, Fragile Handling, Pickup Service | between 0 and 2 |
| Tracking | Tracking Package | exactly one (1..1) |
| Notifications | Notification Package | optional (0..1) |

Cross-group rules: IF International Express selected THEN NOT Standard Cargo Insurance.

Worked validations:
| Selection | Outcome | Rule that decided |
|-----------|---------|-------------------|
| Domestic + Standard + Tracking Package(System) | valid | all groups satisfied |
| International + Standard + Tracking Package(System) | rejected | if international then not standard |
| International + Extended + Tracking Package(Active) | valid | conditional rule only forbids Standard |
| Domestic + Standard + Cold Chain + Fragile + Tracking Package | valid | add-ons = 2, at the max |
| Domestic + Standard + Cold Chain + Fragile + Pickup + Tracking Package | rejected | add-ons = 3 > 2 |
| Domestic + Standard, no Tracking Package | rejected | Tracking 1..1, got 0 |

The outer package checks that exactly one Tracking Package is present; whether that Tracking Package
itself holds exactly one option is checked when the Tracking Package instance is built.

## Boundaries & Integration
- Product module owns: all definitions above, the two catalog entries, the relationship, the
  applicability rules, the three package structures and the validation of shipments and orders.
- Pricing consumes weight, size, scope, chosen insurance and add-ons to compute the price and surcharges.
- Fulfillment owns shipment status; inventory / operations own vehicle and space planning.
- Party supplies customer_type for the pickup rule; ordering assembles the Transport Premium instance and
  asks the product module to validate it.

## Implementation Notes
- Alternative units not needed: every definition has exactly one unit.
- Convention: cardinality counts quantities; two Cold Chain units would count as 2 add-ons.
- Assumed: identical definitions reject supplied serials or batches (strict); size defaults to M.
```

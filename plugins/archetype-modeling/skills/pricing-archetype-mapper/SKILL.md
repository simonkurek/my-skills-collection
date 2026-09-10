---
name: pricing-archetype-mapper
description: Transform domain requirements into a Pricing (valuation) Archetype model. Identifies complexity level (1–9), designs the Calculator layer, Component tree with stakeholder breakdown, Validity versioning, Applicability conditions, and context parameters. Produces an implementable model with explicit concept mapping, unmapped concepts, and integration boundaries.
argument-hint: "[domain requirements or feature description]"
---

# Pricing Archetype Mapper

Transform any domain where a **value is computed according to rules** into a structured pricing model. The value does not need to be a customer price — it can be a fee, interest, commission, tax, surcharge, loyalty points, scoring, or any amount with a unit that depends on context.

> When you hear *pricing*, think *value*. The archetype is really the **valuation archetype**: the organization's generic ability to compute value according to rules, in time, with an explainable result.

**Output goal**: A complete, implementable model that gives the system one source of truth for computed value, a breakdown that shows who each part belongs to, reproducibility of past results, and rule changes that are cheap for the business.

## When to Use

**Use this skill when:**
- A domain requires computing a price / rate / value (not just storing it)
- The computed value depends on context: time, quantity, customer segment, channel, product parameters
- One amount must be split among stakeholders (supplier, operator, partner, tax) or shown as line items
- The value has a temporal lifecycle — definitions change ahead of time, old transactions must remain reproducible
- Several price lists may be valid at once and the system must decide which applies
- The same value is shown in several channels and must agree everywhere
- Rule changes come from the business, not from IT

**Listen for these phrases** — pricing rarely appears under its own name: *"how do we compute this?"*, *"why did it come out differently?"*, *"since when does this apply?"*, *"who is this owed to?"*, and words like rate, discount, limit, interest, penalty, surcharge, recalculation, "applies from".

**Output is useful for:**
- Pricing / valuation module design before implementation
- Multi-stakeholder settlement systems (marketplace, e-mobility, logistics, B2B, regulated industries)
- Deciding whether pricing should be extracted from the product catalog into its own module

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the pricing archetype does not fit, and briefly explain why.

### The core question

> *"Can I ask 'how much is X worth for customer Y at time T in context C?' and get a reproducible, explainable answer with a breakdown?"*

If **yes** → pricing archetype likely fits.
If the natural question is **"how much of X does Y have?"** → it's an accounting ledger. Use `accounting-archetype-mapper` instead.
If the natural question is **"what state is X in?"** → it's a state machine. Do not map.
If the value is a **constant attribute** that never varies and needs no breakdown → level 1; storing it on the product is a conscious, legitimate decision.

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "price depends on quantity / time of day / weight / customer tier" | ✅ Yes |
| "first N units free, then X per unit", "tiers", "over the limit add Y" | ✅ Yes |
| "23% VAT on net", "partner gets 10% of the base", "commission on the amount" | ✅ Yes |
| "new tariff applies from the 1st", "promo valid in February" | ✅ Yes |
| "need to explain why this amount was charged" | ✅ Yes |
| "compute loyalty points / scoring / interest from these parameters" | ✅ Yes — value with a unit, not necessarily money |
| "user earns / spends / transfers N units", "balance" | ❌ No — accounting archetype |
| "order moves from placed → paid → shipped" | ❌ No — ordering / state machine |
| "which accounts do we post VAT and commission to" | ❌ No — accounting; pricing knows *where value came from*, accounting *where to book it* |
| "aggregate the month's usage and issue one invoice" | ❌ No — billing; billing orchestrates pricing, it is not pricing |
| "price is a single stored number, never computed, never changes" | ⚠️ Level 1 only — may not need the archetype |

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The pricing archetype models values computed by rules that depend on context. This domain is a
[accounting ledger / state machine / catalog / billing process / ...] because:

- [specific reason from the requirements]
- The natural question is "[...]" not "how much is X worth for Y at time T?"
```

Do NOT suggest alternative patterns. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — what is being computed, what factors affect it, who receives parts of it, and what changes over time?"

---

### Step 1: Assess Complexity Level

Locate the **highest applicable level** in the requirements. Higher levels include all lower levels. Model **to that level, not past it**.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 1 | **Static value** | One stored number, no context dependency, never changes | (none — adequate for the simplest reality) |
| 2 | **Currency-aware** | Multiple currencies or arithmetic correctness required (`Money`) | one column per currency |
| 3 | **Time-dependent** | Price depends on time of day / date / season | `price_from_hour`, `weekend_flag` columns |
| 4 | **Parameter-dependent** | Price depends on quantity / weight / segment / channel | a growing pile of `if`s |
| 5 | **Stakeholder breakdown** | One amount is split: net, markup, VAT, commission, partner share | the split is done by hand in accounting |
| 6 | **Future-dated change** | "From Monday the new price applies"; old definition must not be overwritten | `UPDATE` on the row |
| 7 | **Value history** | "Why did I pay 142.70?" | event log of values |
| 8 | **Reproducible past** | Old events are re-priced by the rules active at event time — history of *instructions*, not of numbers | feature toggles + a second implementation |
| 9 | **Applicability + consistency** | Several tariffs valid at once; the system selects which applies; every channel shows the same value | `if` order decides; each channel computes its own |

**Guidance — which steps to run:**
- Levels 1–2: The archetype is overkill. Document the level, recommend a value (with `Money` at level 2) on the product, and stop after Step 3.
- Levels 3–5: Core archetype — Calculators (Step 4) + Component tree (Step 5) + Parameters (Step 8). Time dependence at level 3 is a *range* inside a calculator, not versioning.
- Levels 6–8: Add Validity & Versioning (Step 6).
- Level 9: Add Applicability (Step 7) and the eligibility layer above the engine (Step 9).

Partial adoption is normal. Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps. Ask about **two categories** in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Always include **"To zależy / It depends"** as an explicit last option in every question.

#### Category A — Standard pricing decisions

Ask only about those **not clearly addressed** in requirements:

- **Breakdown**: Who needs to see the parts — customer invoice, accounting, partner settlement, regulator? Which parts?
- **Interpretation**: Is the business output TOTAL only (how much does N cost?), or also UNIT (average per unit) and MARGINAL (cost of the N-th unit)?
- **Event time**: Which moment is "the event" for pricing — order, pickup, delivery, session end, settlement? (Decides which side of a rate change an event lands on.)
- **Historical reproducibility**: Must old events be re-priceable using the rules active at event time?
- **Applicability conditions**: Are there business conditions determining whether a component applies, beyond time validity? (segment, channel, region, campaign)
- **Rounding**: To how many places, at which step (per line, per tier, at the end), which mode? Pricing owns the arithmetic; the question is *where*.
- **Product–pricing mapping**: Price stored on product (1:0), one tree per product (1:1), several tariffs per product (1:N), one price for many products (N:1), or independent lifecycles (N:M)?

#### Category B — Gap-triggered questions

Scan the requirements for anything the archetype supports but requirements do not mention:

- **Stakeholder split**: The requirement gives one number — does anyone need to know how it splits among supplier / operator / partner / tax?
- **Dependent components**: Is anything computed *from the value of another part* (VAT on net, commission on base, discount on total)?
- **Structure change over time**: Does the *composition* change (a fee added from May), not just the rates?
- **Recurrence**: Does "summer surcharge" repeat every year, or apply once?
- **Boundary values**: Adjoining bands that repeat a boundary ("1–5 kg", "5–30 kg") — which band is exactly 5 kg in?
- **Undefined inputs**: For lookup price lists (5 / 10 / 20 lessons), what happens for a value not on the list — error, nearest, interpolate?
- **Parallel rate tables**: Same structure with different rates per zone / country / partner?
- **Billing period split**: If a rate changes mid-period, are events priced individually by their own time, or is the period split?
- **Multi-currency**: Components in different currencies? Conversion?
- **Non-money values**: Should the same model compute points / scoring / limits with a unit instead of money?
- **Any other gap** you identify between what the archetype can model and what the requirements specify.

Collect answers before proceeding. If the user cannot answer, document the assumption in **Implementation Notes**. If `AskUserQuestion` is unavailable (non-interactive run), list the questions in prose, pick a reasonable default, and mark it (X).

#### Handling "it depends / both / varies by situation" answers

Always include **"To zależy / It depends"** as an explicit option in every `AskUserQuestion` call — do not rely on the automatic "Other" fallback. Place it as the last option. If the user selects it, the variability itself becomes part of the model:

- Varies by **context** (segment, channel, condition) → an Applicability condition on a component version (Step 7)
- Varies by **time** → separate component versions with Validity (Step 6)
- Varies by **a numeric or time input** → a piecewise calculator over ranges (Step 4)
- Varies by something pricing cannot see (order status, approval, customer history) → computed **above** pricing and passed in as a parameter; note it in **Implementation Notes**

Do **not** model process logic inside the pricing engine.

---

### Step 3: Map Domain Concepts to Pricing Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Pricing Archetype | Notes |
|----------------------|-------------------|-------|
| [domain noun/verb]   | Computed Value / Parameter / Calculator / Interpretation / Component / Composite / Dependency / ComponentVersion / Validity / Applicability / Eligibility / Consumer (outside pricing) | [why] |
```

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear pricing archetype equivalent:
- [concept] — [reason / which module owns it / decision needed]
```

This section must be present even if empty (`None identified`). Typical unmapped concepts: order statuses (ordering), balances (accounting), invoice cycles (billing), bundle definitions (product catalog), rule precedence (rules engine).

---

### Step 4: Design Calculator Layer

Identify which **function shapes** are needed and their parameters.

**Calculator** = pure function `calculate(Parameters) → Money`. It computes a value and only a value — like `sin(x)`. No business conditions, no time validity, no segment logic, no knowledge of what the number means. That belongs in Applicability, Validity and Components.

**Function shapes:**

| Shape | Formula | Use when |
|-------|---------|----------|
| **Fixed** | `f = c` | Flat fee, start fee, per-session charge |
| **Per-unit rate** | `f(q) = rate × q` | Per kWh, per minute, per kg (a fixed calculator read as UNIT) |
| **Percentage** | `f(base) = base × rate` | VAT, commission, margin, discount, insurance — meaning comes from the component name |
| **Step function** | `f(q) = base + ⌊q / step⌋ × increment` | "+X per each started Y units"; note: the staircase never caps |
| **Lookup table** | `f(key) = table[key]` | Bundles defined only for listed values; undefined key → error |
| **Linear over time** | `f(t) = start + elapsed × slope` | Pre-sale ramps, daily increments, interpolation between two points |
| **Piecewise (ranges)** | `f(x) = branch(range containing x)(x)` | Tiers by weight / amount / hour of day / season; any tariff with a flat top tier; ranges are half-open `[from, to)`, may cross midnight, must not overlap |

**Interpretation** — the same function can be read three ways; interpretation is *configuration*, not a separate class:

| Interpretation | Answers | Notes |
|----------------|---------|-------|
| **TOTAL** | what will the bill be for these parameters | default; the only meaningful one for multi-dimensional functions (GB × hours × region has no "unit") |
| **UNIT** | average price per unit at volume N | marketing, comparison |
| **MARGINAL** | price of the N-th unit | tiered tariffs, upsell incentives; the most informative |

Conversions between perspectives are done by adapters, exact only up to rounding. If the business rounds per tier or adds minimum fees, a derived marginal is a mathematical artifact, not a real rate.

**Heuristic**: the most readable representation of a tariff is its chart — draw `f(quantity)` or `f(time)` in words before picking a shape, and check it matches the requirement across the **whole** domain, not just at one sampled point.

**For each Calculator, define:**
- Stable identifier and business name
- Shape and definition-time parameters (rate, step, thresholds, ranges)
- Which call-time parameters it reads from `Parameters` (quantity, base amount, time)
- Interpretation

---

### Step 5: Design Component Tree

Map the value structure as a tree of **Simple Components** (leaves) and **Composite Components** (nodes). This is the layer that carries **semantics**: calculators are math, components give the numbers meaning and an owner.

**Simple Component** — semantic leaf:
- Business name and the stakeholder it belongs to (supplier, operator, partner, tax authority, own margin)
- Reference to one calculator, plus **parameter mappings** from domain names to calculator names (`minutes → quantity`, `kwh → quantity`)
- Interpretation adaptation, so one leaf can be UNIT and its sibling MARGINAL
- Examples: `start-fee`, `energy-cost`, `operator-markup`, `vat`

**Composite Component** — semantic node:
- No calculator; aggregates children, evaluated in declared order, results summed
- Owns the **dependencies** between its children, so children stay reusable in other tariffs:
  - `value of (component)` — use a sibling's computed value
  - `sum of (components)` — e.g. VAT base = sum of net components
  - `difference of (a, b)`, `product of (component, factor)`
- A child may depend only on siblings computed **before** it; to feed a deeper descendant, pass the value down as a parameter
- Examples: `net-cost`, `operator-markup`, `total-session-price`

**Breakdown** — the result tree mirrors the component tree with a value at every node. It is the answer to *"who gets how much"* and the basis for invoices, partner settlements and accounting. It does not by itself say *why* a part is zero; if the business needs that, return the evaluated conditions alongside it.

**Rules:**
- Calculators never return a breakdown; the tree is the breakdown
- Do not confuse semantic composition (a composite: every applicable child fires, results summed) with mathematical composition (a piecewise calculator: exactly one branch fires)
- Discounts, surcharges, taxes and commissions are ordinary components; their meaning is the name, their math is a calculator

**For each component, specify:**
- ID, business name, owner / stakeholder, type (Simple / Composite)
- For Simple: calculator + parameter mappings + interpretation
- For Composite: children in order + dependencies

---

### Step 6: Define Validity & Versioning

If complexity level ≥ 6, components need temporal versioning. (Level 3 time dependence — day/night, season — is a *range* inside a calculator, not a version.)

**Validity** = half-open interval `[validFrom, validTo)`: `always`, `from(t)`, `until(t)`, `between(t1, t2)`.

**Component Version** = immutable snapshot of a component's *details*, with `validity` and `definedAt` (when it was recorded):
- Simple version: calculator + parameter mappings + applicability + validity
- Composite version: children + dependencies + applicability + validity
- The component itself keeps a **stable identity**; only its list of versions grows. Otherwise every rate change repoints every tariff, offer and product that references the component (the domino effect).

**Version selection**: the `timestamp` parameter selects the version valid at that moment. If several are valid (overlay allowed), the one with the latest `validFrom` wins, then the latest `definedAt`. *Time selects the version, not application logic* — there is no `if February`.

**Overlap policy** (decide per component):
- **No overlaps** — strict, fully deterministic partition of time
- **Overlay** — a bounded promo version on top of an open-ended base version; when it expires, *"we do nothing"* and the base applies again
- **Anything** — A/B tests, migrations

**Rules:**
- Versions are append-only. Never update a rate in place; a correction is a new version with the right validity
- A change of **structure** (fee added from May, dependency rewired) is a new version of the **composite**; children keep their own histories
- If a new child must be covered by VAT or a commission, add it inside the composite that is the dependency's base, or rewire the dependency
- Validity is not recurring: "every summer" needs one version per year or a scheduling layer above pricing
- History follows from the model, not from logs: to reproduce the past you need the history of instructions (rates, structure, conditions), which versions give you

**For each versioned component, specify:**
- Overlap policy
- Current version's `validFrom` / `validTo`
- How promotions end: bounded version on a base, or explicit successor version

---

### Step 7: Define Applicability Conditions

If complexity level ≥ 9 (or any component fires only for some contexts), define **Applicability** per component version.

Every component version answers three orthogonal questions:
- **Calculator** → *how* do we compute (math)
- **Applicability** → *whether* we compute at all (segment, channel, context, business condition)
- **Validity** → *when* this definition applies (time)

**Evaluation logic:**
- A simple component version is active when `validity.isValidAt(t) AND applicability.isSatisfiedBy(context)`
- A composite's rule is a **design decision**: either the composite is active when valid and *at least one child* is active (aggregate), or the composite carries its *own* condition that gates the whole subtree (all-or-nothing). State which one you chose
- Conditions are declarative: equals, in set, greater / less than, between, and / or / not. A missing parameter never satisfies a condition
- Non-applicable component → contributes zero. Whether it appears in the breakdown as a zero line or is omitted is a business decision

**Common applicability dimensions:** customer segment (B2C / B2B / VIP), sales channel, region, promotional context, cargo or usage type, thresholds on inputs ("sessions longer than 10 minutes").

**Function vs. condition — the decisive heuristic** (for thresholds like "first 10 minutes free"):
- Threshold stable, changes at most yearly → keep it in the mathematics (range or step)
- Threshold changes per campaign, or depends on segment / channel / country / time of day → applicability condition
- Threshold disappears and returns seasonally → condition **plus** validity periods
- Never encode business concepts (segment B2C as numeric range 0–1) into a function's domain: *the mathematics must not see who the customer is*

The problem is never the math; it is the number of rules, the frequency of their change and the combinatorics of contexts. Rules exist so that mathematics is not forced to pretend to be business policy.

**Scope limit**: pricing holds simple declarative conditions. Precedence, conflict resolution and rule authoring workflows belong to a rules engine — leave them Unmapped with a pointer.

**For each component with applicability, specify:**
- Condition dimensions and logic
- Behavior when not applicable (zero line vs. omitted)

---

### Step 8: Define Parameters & Context Dimensions

Every computation receives a `Parameters` object. Define all dimensions.

**Always mandatory:**
- `timestamp` — the time of the priced **event**, not of the computation; selects component versions. Without it a late event is silently priced by today's rules

**Domain-specific (detect from requirements):**

| Dimension | Purpose | Example |
|-----------|---------|---------|
| `quantity` / `kwh` / `minutes` / `weight` | Input to calculators, mapped by components | `18.2 kWh`, `34 min` |
| `base amount` inputs | Percentage calculators fed from outside (COD value, insured value, loan principal) | `1500 PLN` |
| `customer_segment` | Applicability | `B2C`, `B2B` |
| `channel` | Applicability | `web`, `app`, `pos` |
| `region` / `zone` | Applicability or parallel rate tables | `PL`, `zone-2` |
| `product_id` | Links to product–pricing mapping | `ac-normal` |
| `currency` | Multi-currency models | `PLN`, `EUR` |

For each parameter record its **source** (ordering, billing, product configuration, customer profile) and its **role** (math, applicability, range selection).

---

### Step 9: Determine Product–Pricing Mapping and Integration Boundaries

**Product ↔ Pricing.** Both are versioned trees with conditions — which is exactly why the similarity is dangerous. Product answers *what we sell* (rare, expensive, communicated changes); pricing answers *how much it is worth today, for this customer, in this context* (frequent, cheap, invisible changes).

| Scenario | Structure | When to use |
|----------|-----------|-------------|
| **1:0** | Price stored directly on the product | Low variability, one owner for product and price, no breakdown — conscious simplicity |
| **1:1 mirror** | Each catalog element is a price component | Utilities, telco, e-mobility |
| **1:N** | One product → many price components or tariffs | Banking, logistics, cloud: fixed + variable + surcharges + commissions |
| **N:1** | Many catalog elements → one price | SaaS, insurance, memberships: one holistic pricing decision |
| **N:M** | Independent lifecycles; pricing-only elements (discounts, taxes, regulatory fees, loyalty) with no catalog counterpart | Mature pricing |

Separate pricing from products when prices change more often than products, different people own them, or pricing needs elements that are not products. The decisive argument is organizational, not technical.

**Eligibility belongs above the engine** — selecting *which tariff* applies to a customer needs the customer, their history, campaigns and channel. The application layer picks the root component; the engine computes.

**Integration boundaries** (state them explicitly):

| Concern | Belongs to | Relationship to pricing |
|---------|-----------|-------------------------|
| Order lifecycle, when to price a line | Ordering | Calls pricing per line; pricing never knows order status |
| Aggregating usage over a period, invoices, credits, limits | Billing | Orchestrates pricing per event or sub-period; **never computes value itself** |
| Where to book VAT, commissions, margins | Accounting | Consumes the breakdown; pricing knows where value came from, accounting where to book it |
| Rule precedence, conflict resolution | Rules engine | Pricing holds simple conditions only |
| Balances that accumulate (points balance, credits) | Accounting | Pricing computes *how many* are awarded; the ledger holds them |

Pricing is an autonomous module with its own model, persistence and API — an upstream capability for ordering, billing, quoting, loyalty and analytics. A shared library has no memory (cannot answer *why*), couples deployments and has no owner.

---

### Step 9.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision in the draft model and verify each has a source:
- **(R)** — explicitly stated in requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred by fitting the model to sample outputs given in the requirements (e.g. a rounding policy that reproduces known totals)
- **(X)** — neither: assumed silently

**Decision checklist:**

| Decision area | Example decisions to check |
|---------------|---------------------------|
| Complexity level | Which of the 9 levels? Which layers are deliberately skipped? |
| Computed value | Money or another unit? One value or several (gross / net / partner share)? |
| Event time | Which instant is `timestamp`? |
| Calculator shape per component | Fixed / percentage / step / lookup / piecewise? Does it match the tariff across the whole domain? |
| Interpretation | TOTAL only, or UNIT / MARGINAL too? |
| Dependencies | VAT / commission base = which components? Children ordered accordingly? |
| Rounding | Where, how many places, which mode? |
| Overlap policy | No overlaps / overlay / anything? |
| Applicability | Threshold as function or condition? Composite rule: aggregate or all-or-nothing? Zero line or omitted? |
| Boundary behavior | `>` or `≥` at edges? Which band is exactly 5 kg in? |
| Recurrence | Seasonal rules repeat yearly? |
| Product–pricing mapping | 1:0 / 1:1 / 1:N / N:1 / N:M? Who leads the relation? |
| Orchestration | Ordering per line, or billing per period? |

**For every (X) decision found:**
1. If low impact (purely technical, easily changed): mark as explicit assumption in Implementation Notes.
2. If it affects business behavior (which parts fire for whom, what VAT is computed on, what happens at a version boundary): **stop and ask** using `AskUserQuestion` before delivering the model.

---

## Output Format

```markdown
# Pricing Archetype Model: [Domain Name]

## Pricing Domain
[What is being computed, its unit, detected complexity level (1–9), justification, stakeholders receiving a share]

## Concept Mapping

| Domain Concept | Pricing Archetype | Notes |
|----------------|-------------------|-------|
| ...            | ...               | ...   |

## Unmapped Concepts
[List with owning module, or "None identified"]

## Clarifying Questions & Answers

| Question | Answer | Source |
|----------|--------|--------|
| [question] | [answer / assumption] | R / A / I / X |

## Calculator Design

| Calculator ID | Shape | Definition parameters | Call-time parameters | Interpretation | Notes |
|---------------|-------|----------------------|----------------------|----------------|-------|
| [id] | [shape] | [rates, steps, ranges] | [quantity, base] | TOTAL/UNIT/MARGINAL | [purpose] |

## Component Tree

[ASCII tree with owner per node]

| Component ID | Type | Owner | Calculator / Children | Parameter mappings | Dependencies | Notes |
|-------------|------|-------|----------------------|--------------------|--------------|-------|
| [id] | Simple/Composite | [stakeholder] | [calculator or child list] | [domain → calc] | [algebra] | [purpose] |

## Validity Rules
[Omit if level < 6 — say so]

| Component | Overlap policy | validFrom (current) | validTo | Notes |
|-----------|---------------|---------------------|---------|-------|
| [id] | [policy] | [rule] | [rule] | [notes] |

## Applicability Conditions
[Omit if none — say so]

| Component | Condition | Function or rule — why | Non-applicable behavior |
|-----------|-----------|------------------------|-------------------------|
| [id] | [expression] | [reason] | zero line / omitted |

Composite rule: [aggregate / all-or-nothing]

## Context Dimensions (Parameters)

| Parameter | Type | Mandatory | Source | Purpose |
|-----------|------|-----------|--------|---------|
| timestamp | Instant | Yes | [event] | version selection |
| [param] | [type] | Yes/No | [who supplies] | [purpose] |

## Product–Pricing Mapping & Boundaries

**Scenario**: [1:0 / 1:1 / 1:N / N:1 / N:M]

| Product | Pricing Component Root | Notes |
|---------|----------------------|-------|

| Concern | Belongs to | Integration |
|---------|-----------|-------------|

## Interpretation
[Which interpretations are needed; where adapters are required]

## Worked Example
[One concrete parameter set per scenario worth demonstrating → breakdown with numbers → total]

## Implementation Notes
[Complexity level; layers deliberately not modeled; assumptions (X) and inferences (I); rounding; composite rule; edge cases]
```

---

## Common Patterns & Pitfalls

### Pattern: Mathematics Below, Semantics Above, Conditions Beside

The calculator computes a value and only a value. It does not know it is VAT, a margin or an energy cost; it does not know the segment or the date. The **component** gives the value a name and an owner, the **applicability** condition decides whether it fires, the **validity** decides when the definition applies. A segment check inside a formula, or a rate inside an `if`, means the boundary of responsibility has been crossed.

```
Calculator:    calculate(Parameters) → Money        (math only)
Component:     names the value, assigns the owner   (semantics)
Applicability: isSatisfiedBy(context) → boolean     (business conditions)
Validity:      isValidAt(timestamp)   → boolean     (time)
Eligibility:   selectTariff(customer, context)      (application layer)
```

### Pattern: Variability Is Configuration, Not Code

A promo is a version. A B2B exception is a condition. A weight tier is a range. A new fee from May is a composite version. *"It is not the code that changes, but the configuration — that is the essence of the archetype."* When a requirement says "from next month", "only for", "over the limit" or "in the app", reach for the corresponding block — never for a feature toggle, a second implementation, or an `UPDATE`.

### Pattern: Interpretation Is Configuration, Not Class Hierarchy

Anti-pattern: `StepFunctionTotal`, `StepFunctionUnit`, `StepFunctionMarginal` — every function × three perspectives, three implementations of the same math. Correct: one function, configured with an interpretation; adapters convert between perspectives; a facade selects the adapter. **Pricing owns all arithmetic and rounding** — clients never multiply `unit × n` on their side.

### Pattern: The Consumer Orchestrates, Pricing Computes

Pricing never knows the order, the invoice, the account or the process state. Ordering asks per line; billing asks per event or sub-period and aggregates; accounting receives the breakdown and books it. *"Processes stop computing and start orchestrating."*

### Pattern: History Is a Model Outcome, Not a Log

With versioning done right, reproducing a past price is calling the model with the past `timestamp`. The model is its own audit log. Events and logs give a history of *numbers*; settling the past needs a history of *instructions* — rates, structure, conditions — which only versions provide.

*"Luty mija. Nie robimy nic."* — February passes, we do nothing: the promotional version stops being valid and the base version applies again, with zero conditional logic anywhere.

### Pitfall: Column Soup and If Cascades

`promo_price`, `weekend_flag`, `night_rate_start` and sixty spare columns "for extensibility" are the anti-model. So is the cascade of `if`s that charges the time surcharge on total minutes instead of the excess, writes the threshold as `10:01` because nobody knows whether the interval is closed, and applies the night discount by the end hour. Ranges are half-open `[from, to)` — state it once and stop guessing.

### Pitfall: Treating Product and Pricing as One Tree

Same shape, different responsibility, different pace of change, different owners. Merge them only when neither varies much and one team owns both — and say that you chose to.

---

## Quality Checks

Before returning the model, verify:

- [ ] Complexity level is explicitly stated and justified; skipped layers are named as deliberate
- [ ] The computed value has a unit and its stakeholder shares are listed
- [ ] Every calculator is a pure function (no conditions, no time checks, no business meaning embedded)
- [ ] Every calculator reproduces the stated tariff across its whole domain, not just at one sampled point
- [ ] Every Simple component has an owner, one calculator, parameter mappings (possibly none) and an interpretation
- [ ] Every Composite lists children in order; every dependency references an earlier sibling
- [ ] Applicability conditions live in Applicability, not in calculator math; the composite rule is stated
- [ ] Validity rules use `[validFrom, validTo)` consistently; overlap policy is defined per versioned component
- [ ] `timestamp` is in Parameters, mandatory, and defined as the event time
- [ ] Rounding policy is documented
- [ ] Concept mapping table and Unmapped concepts section are present (even if empty)
- [ ] Product–pricing scenario and integration boundaries are identified; nothing in the model reacts to process state
- [ ] Interpretation strategy documented (TOTAL only, or with adapters)
- [ ] The worked example reproduces a concrete number end-to-end through the breakdown
- [ ] All clarifying question answers (or assumptions) are reflected in the model

---

## Example

**Input:** "Stacja ładowania EV pobiera: opłatę startową 2 PLN, stawkę 0.80 PLN/kWh, dopłatę czasową 0.50 PLN/min za każdą minutę po pierwszych 10 minutach, rabat nocny -10% na naliczoną kwotę dla sesji rozpoczętych między 22:00 a 6:00. VAT 23%. Stawki mogą się zmieniać w czasie — stare sesje muszą być przeliczalne wg stawek z dnia sesji. Operator stacji i operator aplikacji rozliczają się z rachunku."

**Detected complexity level**: 8 — multi-component breakdown for two operators, time-of-day condition, versioned rates, reproducible past.

**Output:**

```markdown
# Pricing Archetype Model: EV Charging Session

## Pricing Domain
**What is computed**: gross price of one charging session, Money (PLN).
**Complexity level**: 8 — breakdown across stakeholders, night condition, versioned rates with reproducible past. Level 9 not reached: one tariff per station, no competing price lists.
**Stakeholders**: station operator (start fee, time surcharge), app operator (energy margin — assumed), tax authority (VAT).

## Concept Mapping

| Domain Concept | Pricing Archetype | Notes |
|----------------|-------------------|-------|
| Opłata startowa 2 PLN | Simple component + fixed calculator | Always applies |
| Stawka 0.80 PLN/kWh | Simple component + per-unit rate | `kwh → quantity` |
| Dopłata czasowa po 10 min | Simple component + linear with free allowance | Threshold kept in the math: stable, changes rarely |
| Rabat nocny -10% | Simple component + percentage; applicability on session start time | Depends on sum of the charges above |
| VAT 23% | Simple component + percentage | Depends on value of net |
| Cena końcowa | Composite (root) | net + VAT |
| Zmiana stawki | New component version with validFrom | Overlay policy |
| Przeliczenie starej sesji | Compute with the session's own timestamp | Reproducible past |
| Rozliczenie operatorów | Breakdown tree | Each node has an owner |

## Unmapped Concepts
- Session lifecycle (started / stopped / failed) — session module; it calls pricing at session end
- Monthly invoice to the driver — billing; aggregates session totals
- Payout of operator shares — accounting; consumes the breakdown

## Clarifying Questions & Answers

| Question | Answer | Source |
|----------|--------|--------|
| Which instant is the event? | session start (night condition) for applicability; session end for version selection — assumed session end for both | X |
| Is the night discount on the whole accrued amount or only on energy? | whole amount before VAT | R |
| Rounding? | 2 dp, half up, per component | X |
| Who owns which part? | start fee + time: station operator; energy: app operator | X |

## Calculator Design

| Calculator ID | Shape | Definition parameters | Call-time parameters | Interpretation | Notes |
|---------------|-------|----------------------|----------------------|----------------|-------|
| `calc-start-fee` | Fixed | 2.00 PLN | — | TOTAL | Per session |
| `calc-energy-rate` | Per-unit rate | 0.80 PLN/kWh | quantity | UNIT | rate × kWh |
| `calc-time-surcharge` | Linear with free allowance | 0.50 PLN/min after 10 min | quantity | TOTAL | 0.50 × max(0, minutes − 10) |
| `calc-night-discount` | Percentage | −10% | base amount | TOTAL | Base fed by dependency |
| `calc-vat` | Percentage | 23% | base amount | TOTAL | Base fed by dependency |

## Component Tree

total-session-price (Composite)                         owner: —
├── net-cost (Composite)                                owner: —
│   ├── charges (Composite)                             owner: —
│   │   ├── start-fee      → calc-start-fee             owner: station operator
│   │   ├── energy-cost    → calc-energy-rate           owner: app operator      kwh → quantity
│   │   └── time-surcharge → calc-time-surcharge        owner: station operator  minutes → quantity
│   └── night-discount     → calc-night-discount        owner: —   base = value of (charges)
│         Applicability: session_start_time ∈ [22:00, 06:00)
└── vat                    → calc-vat                   owner: tax authority     base = value of (net-cost)

| Component ID | Type | Owner | Calculator / Children | Parameter mappings | Dependencies | Notes |
|-------------|------|-------|----------------------|--------------------|--------------|-------|
| `total-session-price` | Composite | — | net-cost, vat | — | vat.base = value of (net-cost) | root |
| `net-cost` | Composite | — | charges, night-discount | — | night-discount.base = value of (charges) | |
| `charges` | Composite | — | start-fee, energy-cost, time-surcharge | — | — | discount base |
| `start-fee` | Simple | station operator | calc-start-fee | — | — | |
| `energy-cost` | Simple | app operator | calc-energy-rate | kwh → quantity | — | |
| `time-surcharge` | Simple | station operator | calc-time-surcharge | minutes → quantity | — | free allowance in the math |
| `night-discount` | Simple | — | calc-night-discount | — | — | negative contribution |
| `vat` | Simple | tax authority | calc-vat | — | — | |

## Validity Rules

| Component | Overlap policy | validFrom (current) | validTo | Notes |
|-----------|---------------|---------------------|---------|-------|
| All simple components | Overlay | business launch | open-ended | Rate change → new version; promo → bounded version on top |
| `net-cost`, `total-session-price` | No overlaps | business launch | open-ended | Structure change (new fee) → new composite version |

## Applicability Conditions

| Component | Condition | Function or rule — why | Non-applicable behavior |
|-----------|-----------|------------------------|-------------------------|
| `night-discount` | session_start_time ∈ [22:00, 06:00) | rule: expected to vary by campaign and segment | omitted from breakdown |
| `time-surcharge` | — (allowance is in the math) | function: 10-minute threshold is stable | shows as 0.00 line under 10 min |

Composite rule: aggregate (a composite is active when any child is active).

## Context Dimensions (Parameters)

| Parameter | Type | Mandatory | Source | Purpose |
|-----------|------|-----------|--------|---------|
| `timestamp` | Instant | Yes | session end | version selection |
| `kwh` | Decimal | Yes | charger meter | energy-cost |
| `minutes` | Integer | Yes | session duration | time-surcharge |
| `session_start_time` | Time | Yes | session start | night-discount applicability |

## Product–Pricing Mapping & Boundaries

**Scenario**: 1:1 — one station type maps to one component tree root.

| Product | Pricing Component Root | Notes |
|---------|----------------------|-------|
| `ev-station-standard` | `total-session-price` | Single tariff per station type |

| Concern | Belongs to | Integration |
|---------|-----------|-------------|
| Session start / stop | Session module | Calls pricing at session end with the session timestamp |
| Monthly invoice | Billing | Sums session totals; never recomputes |
| Operator payouts, VAT reporting | Accounting | Maps breakdown nodes by owner to accounts |

## Interpretation
TOTAL for the bill; energy-cost is UNIT and is normalized to TOTAL inside the tree. UNIT average per kWh not needed in current scope.

## Worked Example
Session started 22:30, 18.2 kWh, 34 minutes:
  start-fee        2.00
  energy-cost      18.2 × 0.80          = 14.56
  time-surcharge   (34 − 10) × 0.50     = 12.00     charges = 28.56
  night-discount   −10% × 28.56         = −2.86     net-cost = 25.70
  vat              23% × 25.70          =  5.91
  total-session-price                   = 31.61 PLN
Same session started at 12:00: no discount → net 28.56, VAT 6.57, total 35.13 PLN.
Same night session with energy rate 0.90 valid from 1 March, priced for 15 March: energy 16.38, charges 30.38, discount −3.04, net 27.34, VAT 6.29, total 33.63 PLN. Priced for 15 February: 31.61 PLN — the old version, no code change.

## Implementation Notes
- Level 8: versions carry `definedAt`; old sessions are re-priced by passing the session timestamp
- Overlay policy for simple components so promotions are bounded versions on the base; no overlaps for composites so structure is deterministic
- Night discount modeled as a rule (not a range) because campaigns are expected to vary it by segment
- Time surcharge allowance kept in the math; move to a rule if it starts varying per campaign
- Boundary: minutes > 10, strict — exactly 10 minutes = no surcharge (X)
- Rounding per component, 2 dp, half up (X) — confirm with finance
- Assumption: single currency (PLN)
- Assumption: energy margin belongs to the app operator; the station operator's energy cost share is not modeled (would be a further split of energy-cost)
```

---
name: quantity-archetype-mapper
description: Transform domain requirements into a Quantity Archetype model — typed amounts with units of measure, money as the flagship case. Identifies complexity level (1–5), catalogs quantities and units, defines per-unit precision and rounding, allowed operations, ratios and splits, conversions within a dimension, rates and interpretation, and the boundaries where sibling archetypes consume quantities.
argument-hint: "[domain requirements or feature description]"
---

# Quantity Archetype Mapper

Transform any domain where **a number only means something together with its unit of measure** into a Quantity model. The unit does not need to be a currency — it can be kilograms, pieces, minutes, gigabytes, licences, tickets, loyalty points, or a rate built from them. Money is the best-known special case: a quantity whose unit is a currency.

> A bare decimal is a digit without identity. Only the unit gives it meaning, correctness rules and the right to be added to something else.

**Output goal**: A catalog of the domain's quantities and units, the rules each unit carries (precision, rounding, sign, integrality), the operations the model must allow and refuse, and the boundaries where sibling archetypes (accounting, pricing, ordering, product, inventory) hand quantities to each other — with rounding ownership named.

## When to Use

**Use this skill when:**
- Requirements mention amounts with units: 5 kg, 100 zł, 8 GB, 40 minutes, 3 licences
- Values of the same kind must be added, subtracted or compared, and mixing kinds must be refused
- Precision or rounding depends on the unit (two decimals for PLN, none for JPY, whole pieces)
- A rate combines two units (price per kilogram, cost per gigabyte-hour) or a percentage is applied to a base
- The same value flows as money in one context and as kilograms or gigabytes in another
- Rounding errors, vanishing pennies or "5 000 of what?" reports have been observed

**Output is useful for:**
- Domain modeling sessions before implementation, and as the value-object foundation the other archetype mappers assume

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the quantity archetype does not fit, and briefly explain why.

### The core question

> *"Is there a number that means nothing without its unit — and do business rules (precision, rounding, allowed operations, compatibility) attach to that unit rather than to the number?"*

If **yes** → quantity archetype fits.
If the number is an identifier, an ordinal, a score or a coefficient of a formula → it is a plain number. Do not map.

**Counter-questions that route to a sibling archetype** (the quantity is still present, but the modeling question belongs elsewhere):
- *"How much X does subject S have, with a history of changes?"* → **accounting** (the ledger stores quantities; this skill only defines what a quantity is)
- *"What is the computed price or value of this?"* → **pricing** (quantities go in, money comes out)
- *"What do we offer, and in which unit is it sold?"* → **product** (the product definition owns the preferred unit)
- *"Who holds it, who is allowed how many?"* → **party** (a bounded count on a role is a constraint, not a quantity)

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "5 kg", "100 zł", "8 GB", "40 min" — the unit is always spoken with the number | ✅ Yes |
| "you cannot add euros to zlotys / kilograms to litres" | ✅ Yes |
| "PLN has two decimals, yen has none", "pieces are whole" | ✅ Yes |
| "price per kilogram", "cost per gigabyte-hour", "23% of the net sum" | ✅ Yes |
| "wholesale by the kilogram, retail by the piece" | ✅ Yes — same product, two units |
| "the balance is 5 000 — but 5 000 of what?" | ✅ Yes — the unit is missing from the model |
| "max 15 patients per day", "up to 25 deliveries" | ❌ No — a capacity constraint on a role (party / rules) |
| "step increment 5, slope 0.3, rate 0.23" | ❌ No — a calculator parameter (pricing) |
| "ticket #4711", "priority 3", "score 87" | ❌ No — identifier, ordinal or score without a unit |
| "a bundle of 3 products" | ❌ No — a bundle's only unit is "number of sets" (product) |
| "price depends on GB and hours and time of day" | ⚠️ Borderline — the result is a total; a "unit" for it may not exist |

### Borderline cases — how to decide

- **Result of a multi-parameter function**: "What is the unit price — per GB, per hour, per GB-hour?" If no contract or price list uses the composite unit, the answer is a **total** in money and the composite unit is not modeled. Only model a composite unit that the business actually writes down.
- **A count used as a limit**: "at most 500 kg per day" is a constraint whose *operand* is a quantity. The quantity (500 kg) fits; the limit rule belongs to the archetype that owns the capability.
- **A rate or coefficient**: 23% VAT, 0.15 zł/kWh. The rate is a quantity only if the business names it, stores it, and applies it in more than one place. A coefficient that lives inside one formula is a pricing parameter.

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The quantity archetype requires a number whose meaning and rules come from its unit of measure.
This domain deals with [identifiers / ordinals / scores / formula coefficients / states] because:

- [specific reason from the requirements]
- The natural question is "[what is X's rank / state / id]?" not "how many of which unit?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — which amounts appear, in which units, and what is done with them (added, compared, priced, converted, split)?"

---

### Step 1: Assess Complexity Level

Locate the **highest applicable level** in the requirements. Higher levels include all lower levels. Model **to that level, not past it**.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 1 | **Typed quantity** | An amount with a unit; two of them are added, subtracted or compared | bare decimal column, unit encoded in the column name (`balance_in_pln`) |
| 2 | **Unit rules** | Precision, rounding, integrality or sign differ per unit (PLN 2 decimals, JPY 0, pieces whole, stock never negative) | rounding repeated at every call site, each slightly different |
| 3 | **Ratios and splits** | A percentage or factor is applied to a base; a whole is divided into parts that must add back up | floating-point arithmetic; 1000 = 800 + 192 and 8 zł vanish |
| 4 | **Conversion within a dimension** | The same kind of thing appears in several units (GB and MB, kg and t, PLN and EUR, seconds and billed minutes) | hard-coded factor; exchange rate without a date |
| 5 | **Rates and interpretation** | Units combine (price per kg, GB-hours), or a value depends on several parameters so a unit value may not exist; a result must say whether it is a unit, marginal or total value | one "unit" column meaning different things in different rows |

**Guidance — which steps to run** (Steps 0, 1, 2, 9.5 and 9.6 run at every level; the list says which of Steps 3–9 to add):
- Level 1: Steps 3, 4 and 9. A typed value object with same-unit arithmetic is the whole model.
- Level 2: add Step 5 (unit rules). Money is the flagship: currency-specific precision.
- Level 3: add Step 6 (operations, ratios, splits).
- Level 4: add Step 7 (conversions). The sources name conversion as a precondition ("convert first") and never model it — keep it a separate concern.
- Level 5: add Step 8 (rates, derived units, interpretation).

Partial adoption is normal — the course itself stays with plain money for most of its lessons. Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps. Ask about **two categories** in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Always include **"To zależy / It depends"** as an explicit last option in every question.

#### Category A — Standard quantity decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as design choices:

- **Sign**: May this quantity be negative? Mathematically every quantity has an inverse (−5 kg, −10 zł); the business may forbid it for stock and allow it for money or corrections. (Signed / non-negative / non-negative except corrections.)
- **Precision per unit**: Fixed scale (PLN 2, JPY 0, kWh 3)? Rounding mode (commercial half-up, banker's half-even, always up for a started unit, always down)? Is storage precision the same as display precision? (The sources give only PLN 2 decimals / JPY 0; no rounding mode is sourced.)
- **Integrality**: Which units are whole-number only (pieces, tickets, licences)? Can 0.5 of one exist?
- **Equality**: Are 1.0 kg and 1.00 kg the same value? (Numerically equal vs scale-exact.)
- **Unit set**: Closed registry of allowed units, or open — any unit a product definition declares? Does each measured thing declare one authoritative unit, or a preferred unit plus acceptable alternatives per channel (wholesale kg, retail pieces)?
- **Conversion**: Is it needed at all? Fixed factor (60 s = 1 min) or dated rate (EUR/PLN on the transaction day)? Who supplies the rate?
- **Split remainder**: When a whole is divided into parts, who absorbs the leftover grosz or gram — the last part, the largest, the first?
- **Ratio base**: For every percentage, what is 100% — the original base or the running value after earlier adjustments?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the quantity archetype supports but the requirements do not mention**. Examples:

- **Unit in the name**: a field or column called `amount_pln` or `weight_kg` — should the unit become part of the value instead?
- **"Per" without a denominator**: "price per unit" — per what? Piece, kilogram, minute, session?
- **Multi-parameter formula asked for a unit price**: price depends on GB *and* hours — is a unit price meaningful at all?
- **Totals derived from rounded unit values**: unit price × quantity ≠ total after rounding — which one is the fact?
- **Two units for one thing across channels**: kilograms wholesale, pieces retail; 1 000 g compared with 1 kg — which is canonical, and where is the conversion?
- **Downstream recomputation**: a consumer re-deriving a value the producer already computed (unit from total).
- **Zero vs absent**: is "0 kg" a value or "not measured"?
- **A rounding rule stated for one unit only**: "billed per started hour" is explicit for time and silent for data — does the convention extend to the other units, or is silence "no rounding"?
- **Percentage cascades**: a rate change ripples through net, VAT, commissions — is the chain of bases explicit?
- **Tolerance**: is "equal" exact or within ±0.05 PLN / ±0.5%?

Collect answers before proceeding. If the user cannot answer, document the assumption made in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

If the user selects **"It depends"**, treat it as a **policy the quantity model accepts as input**:
- Document the *parameter* (e.g. `rounding_mode`, `sign_policy`, `conversion_rate_source`, `remainder_policy`)
- Note in **Implementation Notes** that its value is decided by the consumer or a configuration layer and passed in
- Do **not** model the decision logic inside the value object

#### Without `AskUserQuestion`

If the tool is unavailable or the run is non-interactive: choose the most conservative option for each open question (non-negative, fixed scale from the unit's convention, half-up rounding, no conversion, last part absorbs remainder, percentage of original base), list every choice under **Clarifying Questions & Answers** as an assumption, and mark it **(I)** in Step 9.5.

---

### Step 3: Map Domain Concepts to Quantity Archetypes

Concepts the skill uses (never introduce a code identifier later):

| Concept | Meaning | Key rules |
|---------|---------|-----------|
| **Quantity** | An amount together with a unit | Immutable; every operation yields a new quantity; add/subtract only within the same unit; has a zero. Comparison is beyond the sources — decide its semantics in Step 5 and implement it explicitly |
| **Unit** | A named measure: kg, pcs, PLN, GB, min | Declared with symbol and meaning; whether identity is symbol-only or symbol plus meaning is a decision (Step 4) |
| **Dimension** | A family of mutually convertible units: mass, currency, time, data volume | Units of different dimensions are never added; same dimension may be converted |
| **Money** | A quantity whose unit is a currency | Carries currency-specific precision and rounding; the flagship special case |
| **Ratio** | A dimensionless scalar (percentage, factor) applied to a quantity | Never added to a quantity; always applied to a named base; may compose with other ratios |
| **Precision policy** | Scale and rounding mode for a unit or an operation | Decided explicitly; one owner per rounding step |
| **Conversion** | Mapping between units of one dimension | Fixed factor or dated rate; rounding on conversion is itself a policy |
| **Rate** | A quantity per unit of another: PLN/kg, zł/min | Rate × base quantity yields the numerator's unit |
| **Split** | Dividing a quantity into parts | Parts must sum exactly to the whole; remainder policy named |
| **Interpretation** | What a computed quantity means: unit value, marginal value, total | Only totals can be summed; derivations are exact only up to rounding |

**Design decision — one type or several:** model **one generic quantity type for every dimension** (the trainers' position: money is just a quantity whose unit is a currency) or **a specialised type per dimension** (a money type with currency arithmetic, a separate measured-stock type). The reference implementation does the latter, and its two types deliberately disagree on sign: money may be negative, measured stock may not. One generic type when unit rules are uniform and dimensions are many; specialised types when one dimension needs much richer arithmetic or its sign and precision rules differ. Name the choice — it decides whether "add" is one operation or several.

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept        | Quantity Archetype                                        | Notes |
|-----------------------|-----------------------------------------------------------|-------|
| [domain noun/verb]    | Quantity / Unit / Dimension / Money / Ratio / Precision policy / Conversion / Rate / Split / Interpretation | [why] |
```

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no quantity archetype equivalent:
- [concept] — [reason / which sibling archetype owns it]
```

This section must be present even if empty (`None identified`).

---

### Step 4: Catalog Quantities and Units

Determine every **kind of quantity** the domain measures and the unit each is expressed in.

**Detection signals:**
- Nouns that are counted, weighed, measured, timed or priced
- Units spoken aloud with numbers; units hidden in field names, column names or comments
- The unit a product definition declares (its preferred unit) and any alternative units per channel

**For each quantity kind record:** name, unit, dimension (from level 4; otherwise leave blank), typical values (or "not stated"), who produces it (meter, user, catalog, calculation), who consumes it.

**Rules:**
- Every quantity has exactly one unit at a time; a domain may have many quantity kinds in the same unit (energy delivered and energy reserved are both kWh) — keep them as separate kinds if they must never be confused
- Everything sold or stored has a unit, even a unique item ("one piece")
- Rates and ratios are not catalogued here — a rate has a numerator and a denominator unit (Step 8); a ratio has no unit at all (Step 6)

**Design decision:** a unit's identity is its symbol *plus* its meaning, so two measures sharing a symbol (metres vs miles) are different units; the alternative, symbol-only identity, is simpler and adequate where symbols are globally unique (ISO currency codes).

**Convention** (alternative named): keep the unit set **open** — any unit a product definition declares is valid. The alternative is a closed registry, which suits domains with a fixed vocabulary (currencies, SI units) and rejects typos at the boundary.

**Output:** the Quantities & Units table in the Output Format.

---

### Step 5: Define Unit Rules (Level 2+)

For each unit, state the rules that travel with it.

| Rule | Question | Examples |
|------|----------|----------|
| Scale | How many decimals are stored? | PLN 2, JPY 0, kWh 3, kg 3 |
| Rounding mode | How is excess precision removed? | commercial (half up), banker's (half even), always up (billed per started minute), always down |
| Integrality | Whole numbers only? | pieces, tickets, licences, sessions |
| Sign | Negative allowed? | money: yes (refunds); stock: no; corrections: decision |
| Zero | What does zero mean? | "nothing" vs "not measured" |
| Equality | When are two values equal? | numeric (1.0 kg = 1.00 kg), scale-exact, within a tolerance (±0.05 PLN) |
| Display | Is presentation precision different from storage? | a rate stored at more decimals than it is shown with |

**Rule:** rounding is a business decision, not an arithmetic accident. Every place that rounds must be able to name the rule it applies and the owner who decided it.

**Convention:** operations within a unit keep full precision until the owner rounds once (alternative: round at every step to the unit's scale — simpler to audit, drifts more). **Rule:** derived values (unit price from total, marginal from two totals) are exact only up to that rounding.

**This is a design decision:** rules may live **on the unit** (every PLN amount has 2 decimals) or **on the operation** (division states its own rounding). The reference implementations do both; pick one owner per rule and say so.

---

### Step 6: Define Operations, Ratios and Splits (Level 3+)

Write the **allowed-operation matrix** for the domain. The archetype's structural rules:

| Operation | Allowed when | Result unit | Notes |
|-----------|--------------|-------------|-------|
| quantity + quantity, quantity − quantity | same unit | same unit | different units → refuse loudly, never coerce |
| compare, min, max | same unit | — | comparison across units requires conversion first (Step 7) |
| quantity × number, quantity ÷ number | always | same unit | division needs a precision policy |
| quantity × ratio | always | same unit | the ratio's base must be named |
| ratio × ratio | always | ratio | 50% of 20% is 10% |
| ratio + ratio | same base | ratio | two 10%-of-base discounts add to 20%; two compounding ones do not |
| quantity ÷ quantity, same unit | always | ratio | share of a whole |
| quantity × quantity, quantity ÷ quantity, different units | Level 5 only | derived unit | see Step 8 |
| negate, absolute value | per sign policy | same unit | |

**Ratios**
- A ratio is not a quantity: it has no dimension and is never added to one
- Every ratio application names its **base**: 23% *of the net sum*; 10% *of the original price* vs *of the running price* — the same rate gives different results (two 10% discounts from base: 100 → 90 → 80; compounding: 100 → 90 → 81)
- **Design decision:** model the ratio as a first-class value (composable, typed, stored) or as a plain parameter of the consuming calculation. First-class when the same rate is reused across contexts or composed; parameter when it lives inside one formula.

**Splits**
- **Rule:** the parts of a split sum exactly to the whole. 1000 = 800 + 192 loses 8 zł; the model must either book the remainder or refuse the operation
- Name the **remainder policy**: last part absorbs, largest part absorbs, first part absorbs, or spread by largest remainder
- Splitting by ratios (VAT per component, commission per partner) rounds once per part — so the sum of parts rounded separately may differ from the whole rounded once. State which is the fact (see the Example)

---

### Step 7: Define Conversions (Level 4+)

Only when the same dimension appears in more than one unit.

For each dimension with several units:

```
Dimension: [time / mass / currency / data volume]
  Units:          [s, min, h]
  Canonical unit: [the unit stored and compared]
  Conversion:     fixed factor (60 s = 1 min) | dated rate (EUR→PLN as of transaction date, source: …)
  Rounding:       [policy applied on conversion, e.g. billed per started minute → round up]
  Applied where:  [at the boundary once | on demand for display | never — stored in both]
```

**Rules:**
- Convert before adding; the model refuses to add across units (the sources' only stated conversion rule)
- **Design decision (beyond the sources — none of them models conversion):** a fixed factor is a constant and needs no history; a dated rate is a fact with its own validity, and if you adopt one, record which rate was applied so past results can be reproduced. The alternative the sources effectively take is one unit per dimension, stored and never converted
- Converting changes precision, so it is a rounding step with the same owner and policy obligation as any other (Step 5)

**Design decision:** conversion lives **inside** the value object (it knows its dimension and converts on request) or **outside** as a separate service that the boundary calls. The reference implementations do the latter implicitly: every aggregate works in one unit and never converts. Prefer outside unless conversion is pervasive.

If the domain never converts, write *deliberately not modeled — one unit per dimension* and move on.

---

### Step 8: Define Rates, Derived Units and Interpretation (Level 5+)

**Rates**
- A rate is a quantity per unit of another quantity: 0.15 zł/kWh, 0.10 zł/min
- Rate × base quantity → quantity in the numerator's unit (0.15 zł/kWh × 12 kWh = 1.80 zł); the base's unit must match the denominator
- List every rate the business writes down with its numerator unit, denominator unit and precision
- A tiered rate says whether it applies to the **excess above its threshold** (marginal: "3 zł for every GB above 20") or to the **whole base** (unit: "above 30 GB every GB costs 0.50 zł" is ambiguous — ask)

**Derived units**
- Products of units (GB·h, kg·km, man-days) exist mathematically. **Convention:** model one only if a contract or price list actually uses it; the sources call them exotic and note that for multi-parameter functions "the notion of a unit usually does not exist"
- When several parameters drive a value, let the **total** lead; unit and marginal values recede

**Interpretation**
- A computed quantity must say what it is: **unit** (average per unit of N), **marginal** (the n-th unit), or **total** (what is paid). The mathematics is the same; the meaning is configuration
- **Rule:** only totals are summed across lines, components or stakeholders
- **Rule:** a consumer never derives one interpretation from another that the producer already computed — it takes both as fact (a line keeps unit price 14.27 and total 142.70 for 10 pieces without checking the product)
- **Design decision (left open by the trainers):** carry interpretation as a **tag on plain money** (simplest; enough when only totals matter) or as a **typed result** that carries its unit — unit value as money per unit, marginal value as money plus unit plus index, total as money. Choose typed results when unit safety or stronger type control is needed

---

### Step 9: Define Integration Boundaries

Quantities are consumed by every other archetype. For each boundary, state which quantity crosses, in which unit, and who owns rounding and unit checks.

| Sibling archetype | Quantity role there | Boundary rule |
|-------------------|---------------------|---------------|
| **Product** | Preferred unit of a product type; batch size | The product definition is the **authoritative unit declaration**; alternative units per channel are declared here |
| **Inventory** | Capacity, stock, reservations | Holds quantities in the product's unit; validates a request's unit against the declared unit *(convention: validate here, not in ordering)* |
| **Ordering** | Line quantity; unit price and total price as received | Does not check unit compatibility (catalog/inventory do); does not recompute unit from total or vice versa |
| **Pricing** | Quantities as inputs; money (or typed results) as output | Unit-agnostic inside; the caller guarantees the unit; pricing **owns rounding** of its results |
| **Accounting** | Entry amount, balance | One unit per account (or per entry with the unit as metadata — decision); a transaction's entries share a unit; the zero element is zero *in that unit*, never a hard-coded currency |
| **Plan vs execution** | Planned vs actual amounts; tolerance | Tolerance is absolute (a quantity) or relative (a ratio of the planned value) — say which |
| **Rules** | Thresholds, discount rates, margins | Compare quantities as quantities, never unwrapped raw numbers |

**Rule:** at every boundary, name the **rounding owner**. Downstream takes the owner's numbers as facts.

**Convention:** commands and messages carry amount and unit as primitives and the boundary constructs the typed quantity once (alternative: transport the typed value itself, coupling the message schema to it). Construction is not validation — the boundary checks only that the unit is well-formed; whether it matches the *declared* unit of the thing measured is decided by the owner named in the table (in the sources: the catalog definition and inventory, never ordering).

---

### Step 9.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:

- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred as the conservative default (non-interactive path)
- **(X)** — neither: assumed silently

| Decision area | Example decisions to check |
|---------------|---------------------------|
| Sign | Which quantity kinds may be negative? Corrections? |
| Precision | Scale per unit? Rounding mode per operation? Storage vs display? |
| Integrality | Which units are whole-number only? |
| Equality | Scale-exact or numeric? Tolerance? |
| Unit set | Open or closed? Who may declare a unit? |
| Ratios | Base named for every percentage? First-class or parameter? Compounding or from base? |
| Splits | Remainder policy? Sum-of-rounded-parts vs rounded-whole — which is the fact? |
| Conversion | Needed? Factor or dated rate? Source? Rounding on conversion? Inside or outside the value? |
| Rates & derived units | Every rate's numerator and denominator units? Tiered rates: excess-only or whole base? Any composite unit actually used? |
| Interpretation | Unit / marginal / total stated for every computed value? Tag or typed result? |
| Boundaries | Rounding owner per boundary? Unit validation owner? Zero element per unit? |

**For every (X) decision found:**
1. Low impact (technical, easily changed): mark as explicit assumption in Implementation Notes
2. Affects business behavior (sign of stock, remainder policy, rounding owner, ratio base): **stop and ask** using `AskUserQuestion` before delivering; without the tool, downgrade to (I) and flag it prominently

Do not deliver the model until all material (X) decisions are confirmed or documented.

---

### Step 9.6: Build the Worked Example

Take one concrete case from the requirements and carry it through every row of the Operations, Conversions and Rates tables, showing each intermediate value, the single rounding step and its owner. Verify every number by hand. End with at least one operation the model refuses and why.

---

## Output Format

```markdown
# Quantity Archetype Model: [Domain Name]

## Quantity Domain
[What is measured, the detected complexity level, the levels deliberately not modeled]

## Concept Mapping

| Domain Concept | Quantity Archetype | Notes |
|----------------|--------------------|-------|

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
[Question → answer or assumption, one line each]

## Quantities & Units

| Quantity kind | Unit | Dimension | Produced by | Consumed by | Typical values |
|---------------|------|-----------|-------------|-------------|----------------|

## Unit Rules   (omit if level < 2 — say so)

| Unit | Scale | Rounding | Integer only | Sign | Equality | Zero means | Display |
|------|-------|----------|--------------|------|----------|------------|---------|

## Operations, Ratios & Splits   (omit if level < 3 — say so)

| Operation | Operands | Result | Policy / owner |
|-----------|----------|--------|----------------|

Ratios: [rate → base → first-class or parameter, or "none"]
Splits: [what is split → remainder policy → which sum is the fact, or "none"]

## Conversions   (omit if level < 4 — say so)

| Dimension | Units | Canonical | Conversion | Rounding | Applied where |
|-----------|-------|-----------|------------|----------|---------------|

## Rates & Interpretation   (omit if level < 5 — say so)

| Rate | Numerator | Denominator | Precision | Interpretation the rate expresses |
|------|-----------|-------------|-----------|-----------------------------------|

Computed results and their interpretation: [which values are unit / marginal / total; which are summed]
Derived units: [used / deliberately not modeled]
Result representation: [interpretation tag on money | typed results]

## Integration Boundaries

| Boundary | Quantity crossing | Unit | Rounding owner | Unit check owner |
|----------|-------------------|------|----------------|------------------|

## Worked Example
[Concrete inputs → every intermediate value → outputs, with the rounding step shown]

## Implementation Notes
[Decision register: every material decision with its (R)/(A)/(I) source; assumptions; layers deliberately not modeled]
```

---

## Common Patterns & Pitfalls

### Pattern: The Unit Is Part of the Value, Not of the Field Name

`balance_in_pln`, `weight_kg`, a `currency` column bolted on later — each pushes the unit out of the value and into convention. The model protects you only when the unit travels with the amount and the addition of 100 EUR to 450 PLN is refused by the type, not by a code review.

### Pattern: Same Model, Different Context

Zlotys, kilograms, gigabytes, licences and tickets are indistinguishable from the ledger's perspective: a value and a unit, an additive group with a zero and an inverse. Model the quantity once and let each context give it meaning. A specialised money type is a legitimate choice (see the decision in Step 3); what is never acceptable is the same type with different rules in different places.

### Pattern: Round Once, at the Owner

One rounding step, by the module that owns the rule; every consumer takes the result as fact (Steps 5 and 9). A unit price rounded to two decimals multiplied back by the quantity will not equal the true total — the total is the fact, the unit price a communication figure.

### Pattern: A Ratio Needs a Named Base

23% of *what*? The same percentage function is VAT, a partner margin, a commission or an insurance premium depending only on the base it is pointed at (Step 6). Without a named base the cascade (net changes → VAT changes → commission changes) becomes untraceable.

### Pattern: Interpretation Is Configuration, Not Class Hierarchy

Unit, marginal and total are three meanings of one mathematical function. Do not build three calculators per function, and do not let one calculator answer all three — tag the result with its interpretation and convert between interpretations explicitly, accepting rounding.

### Pitfall: Inventing Composite Units

Kilogram-kilometres and gigabyte-hours are mathematically fine and commercially rare. When a value depends on several parameters, the honest answer is a total in money; do not manufacture a "unit price" for it.

### Pitfall: Unwrapping to a Raw Number

Comparing `stock > 500` after stripping the unit, or seeding a sum with "zero PLN" regardless of the account's unit, silently reintroduces the bugs the archetype exists to prevent. Compare quantities as quantities; the zero element is zero *in this unit*.

### Pitfall: Assuming Comparison Comes for Free

Declaring a value comparable does not make it so. Ordering, min/max and equality semantics (scale-exact or numeric, tolerance or not) are decisions in Step 5 and must be implemented and tested, not inherited.

---

## Quality Checks

Before returning the model, verify:

- [ ] Complexity level stated; every skipped layer marked *deliberately not modeled*
- [ ] Every quantity kind in the catalog has exactly one unit, and a dimension if level ≥ 4
- [ ] Every unit at level 2+ has scale, rounding mode, integrality and sign stated
- [ ] The operation matrix refuses cross-unit addition and comparison explicitly
- [ ] Every ratio names its base and whether it compounds
- [ ] Every split names its remainder policy and which sum is the fact
- [ ] Every conversion names factor or dated rate, its source, and its rounding
- [ ] Every rate names numerator and denominator units; composite units justified by a real price list
- [ ] Every computed value states its interpretation; only totals are summed
- [ ] Every boundary names a rounding owner and a unit-check owner
- [ ] Concept mapping table and Unmapped Concepts section are present
- [ ] Worked example arithmetic verified by hand, including the rounding step; it exercises every table it depends on and shows one refused operation
- [ ] Every clarifying answer or assumption appears in the model or in Implementation Notes
- [ ] All (X) decisions resolved or documented

---

## Example

**Input:** "An EV charging operator bills a session. The charger reports energy in kWh with three decimals and duration in seconds; time is billed per started minute. Wholesale energy is priced in steps: the first 5 kWh at 0.60 zł/kWh, the next 5 kWh at 0.70 zł/kWh, everything above 10 kWh at 0.80 zł/kWh. Grid fee is 0.15 zł/kWh. The station operator charges 1.50 zł per session, 0.25 zł/kWh and 0.10 zł per minute. The e-mobility provider charges 0.10 zł/kWh and 0.05 zł per minute. VAT of 23% applies to the net sum. The customer sees PLN with two decimals. Sample session: 12.000 kWh, 2 395 s."

**Detected level:** 5 — rates in zł/kWh and zł/min, a seconds-to-minutes conversion, a percentage of a named base, currency precision.

**Output:**

```markdown
# Quantity Archetype Model: EV Charging Session Billing

## Quantity Domain
Energy, time and money for one charging session, combined through per-unit rates and a VAT ratio into a customer total. Level 5. All layers modeled; no cross-currency conversion (single currency).

## Concept Mapping

| Domain Concept | Quantity Archetype | Notes |
|----------------|--------------------|-------|
| Energy delivered (kWh) | Quantity, dimension energy | produced by the meter |
| Duration reported (s) / billed (min) | Quantity, dimension time; Conversion | two units of one dimension |
| zł/kWh, zł/min tariffs | Rate | numerator PLN, denominator kWh or min |
| Session fee 1.50 zł | Money, interpretation total | no denominator |
| VAT 23% | Ratio, base = net sum | applied once |
| Net sum, gross sum | Money, interpretation total | only totals are summed |
| "per started minute" | Precision policy on conversion | round up to whole minute |
| "two decimals for the customer" | Precision policy on PLN | scale 2, half up |

## Unmapped Concepts
- Step thresholds (0–5, 5–10, >10 kWh) and stakeholder breakdown — pricing archetype; the quantity model supplies operands and rules only
- Session identity and lifecycle — ordering / product

## Clarifying Questions & Answers
- Sign of money → signed (refunds exist) (A)
- Rounding mode for PLN → half up (I)
- VAT rounded once on the net sum, not per component → yes (A)
- Energy negative (feed-in)? → no, non-negative (A)

## Quantities & Units

| Quantity kind | Unit | Dimension | Produced by | Consumed by | Typical values |
|---------------|------|-----------|-------------|-------------|----------------|
| energy delivered | kWh | energy | meter | pricing | 0.001 – 100.000 |
| duration reported | s | time | charger | conversion at boundary | 60 – 36 000 |
| duration billed | min | time | boundary conversion | pricing | 1 – 600 |
| price components, net, gross | PLN | currency | pricing | ordering, accounting | 0.00 – 200.00 |

Rates (zł/kWh, zł/min) and the VAT ratio are not quantity kinds; they are catalogued under Rates & Interpretation. Typical values are not stated in the requirements and are illustrative.

## Unit Rules

| Unit | Scale | Rounding | Integer only | Sign | Equality | Zero means | Display |
|------|-------|----------|--------------|------|----------|------------|---------|
| kWh | 3 | commercial (half up) | no | non-negative | numeric | no energy delivered | 3 decimals |
| s | 0 | none — integer by measurement | yes | non-negative | exact | no session time | not shown |
| min | 0 | always up (started minute) | yes | non-negative | exact | — | whole minutes |
| PLN | 2 | commercial (half up) | no | signed | numeric | nothing owed | 2 decimals |

VAT ratio precision: 2 decimals as written in the tariff; it has no unit and no row here.

## Operations, Ratios & Splits

| Operation | Operands | Result | Policy / owner |
|-----------|----------|--------|----------------|
| kWh + kWh, PLN + PLN | same unit | same unit | full precision |
| kWh + min | different dimensions | refused | — |
| PLN/kWh × kWh | rate × base | PLN | full precision, no rounding yet |
| PLN/min × min | rate × base | PLN | full precision |
| PLN × % | quantity × ratio | PLN | base = net sum; pricing rounds once to scale 2 |
| sum of components | PLN totals only | PLN | pricing |

Ratios: VAT 23% → base: net sum → parameter of the pricing calculation (not reused elsewhere)
Splits: none — VAT is not distributed back to components; the whole rounded once is the fact

## Conversions

| Dimension | Units | Canonical | Conversion | Rounding | Applied where |
|-----------|-------|-----------|------------|----------|---------------|
| time | s, min | min (billed) | 60 s = 1 min, fixed | always up to a whole minute | once, at the meter → pricing boundary; stored in both (raw seconds as evidence, billed minutes as the priced value) |

## Rates & Interpretation

| Rate | Numerator | Denominator | Precision | Interpretation the rate expresses |
|------|-----------|-------------|-----------|-----------------------------------|
| wholesale energy 0.60 / 0.70 / 0.80 | PLN | kWh | 2 | marginal (each step applies to the excess above its threshold) (R) |
| grid fee 0.15 | PLN | kWh | 2 | unit |
| operator 0.25, provider 0.10 | PLN | kWh | 2 | unit |
| operator 0.10, provider 0.05 | PLN | min | 2 | unit |
| session fee 1.50 | PLN | — | 2 | total |

Computed results: energy net 9.90, operator 8.50, provider 3.20, net 21.60, VAT 4.97, gross 26.57 — all **total** interpretation, and only these are summed; the marginal and unit labels above describe the price list, not the results.
Derived units: none — no GB·h-style unit is used by the tariff; deliberately not modeled
Result representation: interpretation tag on money (only totals leave pricing)

## Integration Boundaries

| Boundary | Quantity crossing | Unit | Rounding owner | Unit check owner |
|----------|-------------------|------|----------------|------------------|
| meter → pricing | energy, duration | kWh, s → min | boundary (minute round-up) | boundary |
| tariff config → pricing | rates, VAT | PLN/kWh, PLN/min, % | — | pricing |
| pricing → ordering | component totals, net, gross | PLN | pricing | — (taken as fact) |
| ordering → accounting | gross amount | PLN | none (already rounded) | account unit = PLN |

## Worked Example
Duration: 2 395 s ÷ 60 = 39.9166… min → rounded up once, at the boundary → 40 min (no intermediate rounding).
Energy 12.000 kWh, wholesale (marginal steps): 5 × 0.60 = 3.00; 5 × 0.70 = 3.50; 2 × 0.80 = 1.60 → 8.10 zł
Grid fee: 12 × 0.15 = 1.80 zł → energy net 9.90 zł
Station operator: 1.50 + 12 × 0.25 (3.00) + 40 × 0.10 (4.00) = 8.50 zł
E-mobility provider: 12 × 0.10 (1.20) + 40 × 0.05 (2.00) = 3.20 zł
Net sum (totals only): 9.90 + 8.50 + 3.20 = 21.60 zł
VAT: 21.60 × 23% = 4.968 → rounded once, half up, scale 2 → 4.97 zł
Gross: 21.60 + 4.97 = 26.57 zł
Why the rounding owner matters: VAT per component would be 9.90 × 23% = 2.277 → 2.28; 8.50 × 23% = 1.955 → 1.96; 3.20 × 23% = 0.736 → 0.74; sum 4.98 zł — one grosz more than the fact. The customer pays 26.57.
Refused: 12 kWh + 40 min (different dimensions); 26.57 PLN + 12 kWh.

## Implementation Notes
- Commercial half-up rounding for PLN is an inferred default (I); banker's rounding would change nothing in this session but must be confirmed for volume billing
- Equality: numeric for kWh and PLN, exact for integer units (I)
- Wholesale steps apply to the excess above each threshold (R: "the next 5 kWh", "everything above 10 kWh")
- Energy is non-negative (A); a feed-in scenario would reopen the sign decision
- Duration is stored in billed minutes after conversion; the raw seconds are kept as measurement evidence, never re-billed
- Conversion lives outside the value object, at the meter boundary (convention: no aggregate converts)
- No cross-currency conversion and no composite units — deliberately not modeled
- Accounting posts one PLN entry per session; the zero element of its sums is 0.00 PLN because the account's unit is PLN, not by default
```

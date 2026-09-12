---
name: plan-vs-execution-archetype-mapper
description: Transform domain requirements into a Plan vs Execution Archetype model — a first-class plan of intent, a separate record of what actually happened, and a computed delta between them governed by a tolerance. Identifies complexity level (1–7), the plan and execution sides, tolerance and matching policy, the three-bucket delta with statistics, resolution bridges between granularities, re-planning rules with simulation, and plan history. Produces an implementable model with explicit concept mapping, unmapped concepts and integration boundaries.
argument-hint: "[domain requirements or feature description]"
---

# Plan vs Execution Archetype Mapper

Transform any domain where **something was supposed to happen and something did happen** into a model that keeps the two apart and computes the difference on demand. The plan does not need to be a schedule — it can be a repayment plan, a production target, a budget, a forecast, a delivery promise, a rota, a treatment protocol, or any record of intent that reality is later measured against.

> A plan is a *value* the business committed to; execution is a set of *facts* the world reported; the delta is a *third thing* computed from the two. Never let any of them live inside another.

**Output goal**: A complete, implementable model that gives the system an addressable plan with a history, execution facts that never overwrite it, a delta that can be recomputed for any plan × execution × tolerance combination, and re-planning that is explicit, explainable and simulable.

## When to Use

**Use this skill when:**
- The domain keeps a record of what *should* happen (schedule, target, budget, promise, forecast) and a record of what *did* happen, and someone asks how far apart they are
- Deviations are judged with a tolerance — "within 5 groszy", "up to 10 days late", "±5 %" — and the same facts may be acceptable under one tolerance and not another
- One planned thing may be satisfied by several actual things (partial payments, split batches, deliveries in tranches) or the plan and the execution are recorded at different granularities (monthly target, daily output)
- The business reacts to the delta by changing the plan — restructuring, waiving, adding a buffer — and wants to know why the plan changed and what it was before
- People ask what-if questions: "what if we had promised the 25th?", "what if tolerance were 5 %?", "compare this execution against last month's plan"

**Listen for these phrases**: *on time / late / early*, *short / over*, *within tolerance*, *deviation*, *variance*, *how far off plan*, *re-plan / restructure / re-spread*, *what if*, *what was the plan before*, *why did the promise change*.

**Output is useful for:**
- Designing repayment, production, delivery, budgeting, staffing and SLA-tracking modules before implementation
- Deciding whether a "status" field on an order or task is enough, or whether the comparison deserves its own model

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the plan-vs-execution archetype does not fit, and briefly explain why.

### The core question

> *"Is there a record of what should happen, kept separately from a record of what did happen, and does the business ask how far apart they are and what to do about it?"*

If **yes** → plan-vs-execution archetype likely fits.
If the natural question is **"how much X does S have?"** → it's an accounting ledger. Use `accounting-archetype-mapper`; a plan can feed a ledger (memo accounts, planned entries) but a plan is not an account.
If the natural question is **"who requested what, on what terms, and who executes it?"** → it's ordering. Use `ordering-archetype-mapper`; ordering supplies the *intent* side and routes the comparison here only when deviations, tolerance or re-planning matter.
If the natural question is **"in what order must the steps run?"** → it's scheduling over a dependency graph. That *builds* a plan; it does not compare one with reality.
If the natural question is **"which resource can I allocate?"** → it's inventory.
If the natural question is **"what state is X in?"** → it's a state machine. Do not map.

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "instalments due on the 15th; payments come in whenever" | ✅ Yes |
| "target 300 units this month; daily output is reported" | ✅ Yes — different granularities |
| "paid within 5 groszy and 10 days counts as paid; otherwise late" | ✅ Yes — tolerance |
| "two on-time payments → waive one instalment; one late → restructure" | ✅ Yes — re-planning |
| "what if we had promised the 25th?", "compare against the original plan" | ✅ Yes — simulation, history |
| "budget vs actual spend per cost centre, with variance report" | ✅ Yes |
| "customer's balance / how much is still owed" | ❌ No — accounting; the delta may *post* to it |
| "who ordered, who pays, agreed price, cancellation terms" | ❌ No — ordering |
| "which steps can run in parallel, what depends on what" | ❌ No — scheduling graph (builds the plan) |
| "which slot / driver / warehouse is free" | ❌ No — inventory |
| "ticket open → assigned → resolved" | ❌ No — state machine |
| "delivery status: planned / in transit / delivered" | ⚠️ Borderline — ask whether anyone needs *how far off*, *why it changed*, or *what if* |

### Borderline cases — how to decide

- **Fulfilment status on an order**: The moment the business asks *how late*, *how short*, *within what tolerance*, or *should we re-plan*, the comparison needs its own model. Ordering remains the owner of the intent; this archetype owns the comparison.
- **Budget vs spend**: Accounting can hold a budget as memo entries and allocate spend against it. Stay in accounting if the only question is "how much is left". Come here when items must be matched one to one, judged with a tolerance, and the budget itself is re-planned in reaction.
- **A forecast**: A plan nobody committed to is still a plan if actuals are compared against it and the forecast is revised in response. Fits; note the weaker commitment in Implementation Notes.
- **A promise computed from live data** (SLA days + calendar + capacity): looks like there is no plan at all. There is — it is just never recorded. That is the pathology level 2 fixes (its "naive solution that breaks"), not a no-fit and not a level-1 domain.

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The plan-vs-execution archetype requires a record of intent kept apart from a record of facts,
and a business need to compare them. This domain is a [ledger / order / state machine /
scheduling graph / inventory / ...] because:

- [specific reason from the requirements]
- The natural question is "[...]" not "how far is execution from the plan, and what do we do about it?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — what is planned, what gets reported as actually happening, how deviations are judged, and what the business does when they occur?"

---

### Step 1: Assess Complexity Level

Locate the **highest applicable level** in the requirements. Higher levels include all lower levels. Model **to that level, not past it**.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 1 | **Status flag** | One record with planned and actual fields; nobody asks why it changed or what-if | (none — adequate for the simplest reality) |
| 2 | **Separated plan and execution** | "What was promised must not change when the customer's SLA is edited"; execution must not overwrite the plan | the plan recomputed from live tables on every call; plan and actual columns on one mutable row |
| 3 | **Delta with tolerance** | "Within 5 groszy counts as paid"; "report missing and unexpected"; on-time vs late | a flat enum (`late / under / deviation`) that cannot express two dimensions at once |
| 4 | **Partial and split execution** | One instalment paid in three transfers; one target met by several batches; a deadline for partials | one-to-one join on amount and date |
| 5 | **Resolution mismatch** | Monthly target vs daily output; quarterly budget vs weekly spend | summing whatever rows exist and comparing to the headline number |
| 6 | **Re-planning and simulation** | "If late twice, restructure"; "what if we had promised the 25th?"; several scenarios side by side | editing the plan in place; copies with a `simulation` flag |
| 7 | **Plan history and intent** | "What was the plan before the change, who changed it, why, from when?"; compare execution against any past plan | `last_modified_by` without the *why* or the *before* |

**Guidance — which steps to run:**
- Level 1: The archetype is overkill. Document the level, recommend a plain record with a status, and stop after Step 3. Say explicitly what the business gives up (no history, no what-if).
- Levels 2–3: Core archetype — Plan side (Step 4), Execution side (Step 5), Comparison (Step 6).
- Level 4: Add the collection-valued match and deadline rules inside Step 6.
- Level 5: Add the Resolution Bridge (Step 7).
- Level 6: Add Re-planning Rules and Simulation (Step 8).
- Level 7: Add Plan History in Step 9.
- Every level ≥ 2: run the **Boundaries** half of Step 9 — naming what comes from ordering / product and what is posted to accounting is not a level-7 concern.

Partial adoption is normal. Mark skipped layers in Implementation Notes as *deliberately not modeled*.

A domain may hold **several plan/execution pairs** (a monthly target and a per-run plan; a budget and a delivery schedule). Level each pair on its own, run each step only where that pair's signal appears, and either give the output one column per pair or produce one model per pair — say which.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps. Ask about **two categories** in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Always include **"To zależy / It depends"** as an explicit last option in every question.

#### Category A — Standard plan-vs-execution decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as design choices:

- **Sameness (tolerance)**: What makes an actual item "the same thing" as a planned item — exact equality, an absolute band, a percentage, a number of days, a combination? Is the band symmetric (early as acceptable as late, over as acceptable as under)?
- **Identity gate**: Must planned and actual items share a key (product, contract, obligation, cost centre) before they may be compared at all, or is comparison purely by amount and time?
- **Partial and split execution**: May one planned item be satisfied by several actual items? Is there a cut-off after which contributions no longer count? Conversely, must one actual item be *split* across several planned items? If yes, that split is an allocation decision (which planned item is deemed consumed first: earliest, largest, manual), made when the fact is recorded rather than at read time; note it as a parameter and see the boundary note in Step 9.
- **What counts as done**: Is "actual" a raw reading, or a formula (net of defects, returns, refunds, rework)? Who defines it?
- **Which date decides lateness**: The business date of the event, or the date the system learned about it?
- **Over-execution**: Is more than planned a deviation to report, or silently acceptable? (Carrying a surplus forward as a credit against the next planned item is a *balance*, which belongs to the ledger downstream — see Step 9.)
- **Reaction to the delta**: Does the business re-plan automatically, or does it want a diagnostic signal for a human? Which questions about the delta trigger it, and is each concession granted once or every time?
- **History**: Must the system answer "what was the plan before?" and compare execution against a past plan, or is a single current plan enough?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the archetype supports but the requirements do not mention**. For each gap, ask whether that dimension is wanted. Examples:

- **Granularity**: Are plan and execution recorded at the same grain? If not, how is the coarse side projected onto the fine one, what about a period not yet over, and is the tolerance per item or on the period total? (Detail in Step 7.)
- **Absence**: Must the system react when *nothing* happened (no payment at all), not only to things that happened late or short?
- **Corrections**: Can an execution event be cancelled, reversed or corrected after it was compared? What happens to the delta and to any re-planning it triggered?
- **Settled lines and intent**: Do already-satisfied planned items stay in the plan, may a rule touch them, and must a plan change record its reason, author and effective-from date? (Detail in Steps 8–9.)
- **Duplicate facts**: Are two identical execution events (same amount, same day) two facts or one?
- **Downstream consumers**: Does the delta feed a ledger (penalties, waivers, interest), notifications, or reporting?
- **Scenarios**: Must several what-if plans or tolerances be evaluated against the same execution and kept side by side with a name?

Collect answers before proceeding. If the user cannot answer, document the assumption in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

If the user selects **"To zależy / It depends"**, treat it as a **variable policy supplied from outside the comparison**:

- Document the *parameter* the model accepts (e.g. `tolerance`, `resolution_bridge`, `what_counts_as_done`, `once_only`)
- Note in **Implementation Notes** that its value is chosen by configuration, contract or a rules layer and passed in when the comparison runs
- Do **not** model the decision logic inside the archetype

This is the correct outcome — the archetype computes with a policy; it does not choose it.

#### If `AskUserQuestion` is not available

List the questions in prose, pick a reasonable default for each, mark every such default **(X)** in the sanity check, and carry it into Implementation Notes as an explicit assumption.

---

### Step 3: Map Domain Concepts to Plan vs Execution Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Plan vs Execution Archetype | Notes |
|----------------------|-----------------------------|-------|
| [domain noun/verb]   | Plan / Planned Item / Execution Event / Actual Item / Tolerance / Match / Delta / Statistic / Resolution Bridge / Re-planning Rule / Plan Version | [why] |
```

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear plan-vs-execution archetype equivalent:
- [concept] — [reason it doesn't fit / decision needed]
```

This section must be present even if empty (`None identified`).

**Concept vocabulary used from here on:**

| Concept | Meaning | Key rules |
|---------|---------|-----------|
| **Plan** | An ordered, immutable collection of planned items with its own identity | First-class and addressable; never a query result over other data |
| **Planned Item** | One line of intent: what, how much, and (if the domain has time) when | Immutable value |
| **Execution Event** | A fact reported from outside, with its own identity, a business date and a recording date | Never touches the plan |
| **Actual Item** | The projection of one or more execution events onto the plan's dimensions | May be the same shape as a planned item or a different one |
| **Tolerance** | The rule that decides whether a planned item and a *group* of actual items are the same thing | Composable; answers with a reason |
| **Match** | One planned item bound to one or more actual items, with the reason and the residual deviation | An actual item is bound at most once |
| **Delta** | The first-class result of comparing one plan with one set of actuals under one tolerance | Three buckets: matched, missing, unexpected |
| **Statistic** | A number rolled up over the delta: counts, totals, on-time and late | Judges *quality*; the tolerance judges *identity* |
| **Resolution Bridge** | The explicit rule that projects one side onto the other's granularity | A business decision, injected, never hard-wired |
| **Re-planning Rule** | A condition asked of the delta, bound to a modifier that turns the current plan into a new one | Pure; ordered; may be once-only |
| **Plan Version** | A plan as it was at a moment, with reason and effective-from | Level 7 |

---

### Step 4: Define the Plan Side

Make the plan an explicit artifact.

**Detection signals:**
- A number or date the business "promised", "targeted", "scheduled" or "budgeted"
- A calculation that is redone on every request and whose inputs (SLA, calendar, capacity) belong to other aggregates
- Business people saying "the plan changed" without being able to say when

**For the plan, decide:**
- **Dimensions of a planned item**: which of *what* (identity key), *how much* (amount, quantity), *when* (due date, period) apply. A repayment plan has amount and date and no item identity; a production plan has product and quantity and no time. Both are complete.
- **Ordering**: Is the plan ordered (by due date) and is that an invariant? Time-based plans usually are. Quantity-only plans usually are not — and then note that the greedy walk in Step 6b still consumes actual items in the order they arrive, so the delta is reproducible only if the plan and the candidate pool are given a stable order (by identity key, by arrival) before matching. Say which order that is.
- **Identity of the plan**: Which contract, loan, order, month or cost centre does it belong to? One plan per subject per period is the usual shape.
- **Provenance**: Where do the planned numbers come from — an ordering module (agreed terms), a product definition (repayment plan template), a scheduling graph, a human? The plan *records* them; it does not recompute them. If the requirements are silent, name the owner as an (X) assumption.

**Rule:** the plan is a value. Recording execution never modifies it; re-planning produces a new plan (Step 8). This is the structural condition for every later step.

**Rule:** a plan is never reconstructed from live source data. If the promise depends on a customer's SLA, the warehouse's capacity and a calendar, the *result* is stored as the plan, with a reference to the inputs used. Otherwise editing any input silently rewrites every promise that depended on it, and "what was the plan yesterday" becomes unanswerable.

---

### Step 5: Define the Execution Side

Separate what the world reported from what the comparison needs.

**Detection signals:**
- Events arriving from payment systems, shop-floor terminals, delivery scans, timesheets
- Fields such as "processed at", "recorded on", "reported by" next to the business date
- Several raw readings that together make one comparable number
- The consistency boundary: the plan is where rules and data are protected (strong consistency); execution is where processes reconcile over time (eventual consistency). Where that line falls is where the plan record stops and the execution record starts.

**For execution, decide:**
- **Execution Event vs Actual Item**: Keep the raw event (own id, business date, recording date, source) and derive the actual item from it. When the domain has no raw events (actuals are typed in), the actual item is the event; say so.
- **What counts as done**: *"Actual" is itself a formula.* Net good output = produced − defects + rework; gross output = produced. Money received net of refunds, or gross. Choose explicitly and name the alternative rejected.
- **Business date vs recording date**: Only the business date (when the payment was made, when the batch left the line) drives matching and lateness. The recording date is kept for audit and reconciliation and is never a matching input. Convention: if the domain insists the recording date matters, that is a second comparison (reporting timeliness), not a change to this one.
- **Corrections**: a design decision — either a cancellation or correction is a *new* fact that supersedes the old one and the delta is recomputed from the full history (every past delta stays reproducible; the event set grows), or the event is amended in place and past deltas silently change (cheap; gives up "what did we think last month"). Immutability of the plan is worth little if the facts move underneath it — prefer superseding and say so.
- **Duplicates**: Two events with the same amount and date are two facts if they have their own identity. Give execution events identity precisely so that binding one does not consume the other.

---

### Step 6: Design the Comparison — Tolerance, Matching, Delta

This is the heart of the archetype: a third object computed from the two sides.

#### 6a. Tolerance — deciding sameness

The tolerance answers one question: *is this planned item satisfied by this group of actual items?* It returns yes or no **with a reason**.

| Shape | Formula | Typical use |
|-------|---------|-------------|
| Exact | Σ actual equals planned exactly (and, where the domain has time, every date equal) | audits, penny-perfect reconciliation |
| Absolute band on amount | \|planned − Σ actual\| ≤ a | "within 5 groszy", "±10 units" |
| Percentage band | \|planned − Σ actual\| ≤ p % × planned | "±5 %"; define the zero-planned case explicitly |
| Date window | every actual within d days of the planned date | "up to 10 days" |
| Aggregate before a cut-off | Σ actual *on or before the deadline* within band | partial payments that must be complete by the due date |
| Composition | AND / OR / NOT of the above, reason strings composed | "within 1 grosz AND within 10 days" |

**Rules:**
- The actual side of the tolerance is always a **collection**, even when the domain expects one item. This is what makes partial and split execution (level 4) expressible without a special mode.
- With a cut-off shape, decide what happens to group members that fall *after* the cut-off: consumed by the match (they vanish from the unexpected bucket without contributing to the sum — "the instalment is settled, whatever arrived later") or left in the pool (they surface as unexpected). They are not interchangeable.
- Every verdict carries a human-readable **reason**. "Not matched: amount differs by 0.07 PLN" is part of the model, not a log line.
- **Tolerance decides identity, statistics decide quality.** A payment accepted by a 10-day window is *still late*. A batch accepted by a ±5 % band is *still short by 2*. Never let the tolerance absolve the fact.

**Design decisions to state explicitly:**
- **Exact means what**: does exact matching also require a *single* actual item, or may several sum to the planned amount exactly? Time-bearing domains usually want the singleton rule; quantity-only domains usually do not (three batches summing exactly to the target are exactly right).
- **Symmetry**: the shapes above are symmetric (early as acceptable as late, over as under). Many businesses want asymmetry (early always fine, late never; over fine, under not). Choose and say so.
- **Identity gate**: whether a key (product, obligation) must match *before* the tolerance is consulted. If it must, the gate belongs to the matching policy, not to the tolerance.

#### 6b. Matching policy — pairing items

Walk the plan in its order; for each planned item, find a group of still-unbound actual items the tolerance accepts; bind them.

**Rules:**
- An actual item is bound **at most once**. Once it satisfies a planned item it leaves the pool.
- A planned item binds **one or more** actual items (level 4) — never zero; an unsatisfied planned item is *missing*, not matched-with-nothing.
- Whatever is left on either side after the walk is the leftover bucket. No second pass silently re-assigns.

**Design decision — which candidate wins:** *first fit* (smallest group, earliest position — cheap, deterministic, may prefer a mediocre single item over a perfect pair) or *best fit* (closest amount or date — more work, needs a scoring rule). Name the choice, or write *not applicable* when at most one candidate can ever exist; with a date-blind tolerance and first fit, an instalment can be satisfied by a payment from another month, and that may or may not be acceptable.

**Convention:** cap the group size for split execution (e.g. at most five contributions per planned item) and make the cap a parameter. The alternative is no cap — every group size is tried, which is exhaustive, quadratic, and lets an implausible ten-way split satisfy an item. Cap by default; say which you chose.

#### 6c. Delta — the three buckets

```
Delta( plan P, actuals A, tolerance T ) =
  matched    : [ Match(planned, [actual…], reason, residual) ]
  missing    : [ planned items no acceptable group satisfied ]     ← under-execution
  unexpected : [ actual items no planned item accepted ]           ← over-execution
  statistics : rolled-up numbers over the three buckets
```

**Rules:**
- The delta is a **first-class, immutable, returnable value**. It can be stored, compared, recomputed for any (plan, actuals, tolerance) triple, and read by many consumers. It is not a method on the plan.
- A match is **not** an equality. It keeps the residual (2 units short, 5 days late) and the reason it was accepted.
- Both leftover buckets matter. Over-execution is as much a deviation as under-execution.

**Design decisions to state explicitly:**
- **Perfect**: "no leftovers" (deviations inside the tolerance are fine) or "no leftovers and every residual is zero". The two answers differ for the same facts; default to the strict reading whenever the business reports residuals at all.
- **Gross vs net**: a planned 100 whose only candidate, an actual 150, is rejected by an exact tolerance is *100 missing and 150 unexpected* (gross, both true — "unexpected" means *not accepted*, not *not wanted*) with a *net +50*. Keep the gross buckets and report the net figure alongside; do not net per item silently.

#### 6d. Statistics — judging quality

Roll up whatever the business asks of the delta: counts (planned, matched, missing, unexpected), totals (planned, actual, missing amount, unexpected amount, net difference), timeliness (on time = every bound actual on or before the planned date; late = any bound actual after it), completeness (matched ÷ planned). Note the asymmetry: on-time requires *every* bound actual to be punctual, late needs only one — a single late contribution taints an otherwise timely group, and a group of partials yields one timeliness verdict, not several.

**Design decision — where do accepted residuals go?** A planned item the tolerance accepted but that came in short can be counted two ways, and the answers differ for identical facts:
- **Leftovers only**: the shortfall total is the sum of the *missing* bucket; a 2-grosz shortfall inside a matched item is invisible to it, and "no shortfall" can be reported while money is genuinely owed.
- **Leftovers plus residuals**: each matched item's residual is added to the missing bucket; the same delta then reports a shortfall even when every planned item matched.
Name the choice, make the "is anything under-executed?" predicate agree with it, and apply the same choice to over-execution.

**Convention:** count *facts* and *matches* separately when they differ — three partial payments bound to one instalment are one match and three payments. Say which the business means when it says "how many payments".

---

### Step 7: Bridge Resolution Mismatch

Run this step only at level 5 or above. Otherwise write *"Not applicable — plan and execution share a grain"* in the output.

**Detection signals:**
- "300 units this month" vs a daily production log
- Quarterly budget vs weekly invoices
- A promise per order vs scans per parcel

**The bridge is a business rule, injected, never hard-wired.** Decide:
- **Direction**: project the coarse plan *down* to the fine grain (300 per month → a daily target), or aggregate the fine execution *up* to the coarse grain (daily log → monthly total). Down-projection lets you compare day by day; up-aggregation is simpler and answers only the period question.
- **Projection rule**: evenly over calendar days, over working days, weighted by a profile? Calendar-aware (31-day months, holidays) or not? What happens to the remainder of an uneven split?
- **Partial period**: is a month that is not over compared pro-rata, or only when closed? Is a day with no record "zero" or "not yet reported"?
- **Tolerance level**: per projected item, or on the period total? On the period total, daily swings cancel (50 over one day and 50 under another passes a ±10 monthly band) and the answer is a bare *acceptable / not acceptable*: no matches, no missing or unexpected buckets, no per-item statistics, because nothing was paired. Per projected item gives the full three-bucket delta at the cost of a projection that must be defensible day by day. If you choose the aggregate level, say so in the Delta section instead of filling the bucket table.

**Rule:** different bridges legitimately give different verdicts on identical data, so the bridge belongs to the delta's inputs alongside the plan, the actuals and the tolerance — a delta whose bridge is unknown cannot be recomputed or compared. Illustration: a monthly target of 300 and actual net output of 283 with an aggregate tolerance of 20 — split evenly over 30 days the projected plan sums to 300 and the deviation is 17 (acceptable); split over 22 working days rounding up per day it sums to 308 and the deviation is 25 (out of tolerance). Rounding alone moved the verdict.

---

### Step 8: Define Re-planning Rules and Simulation

Run this step only at level 6 or above. Otherwise write *"Not applicable — the delta is reported, the plan is not changed by the system"*.

A re-planning rule binds a **condition** to a **modifier**:

```
Rule: [name]
  Condition: a question asked of the DELTA   (e.g. "late matches ≥ 1", "on-time matches ≥ 2",
                                              "missing quantity ≥ 50")
  Modifier:  (current plan, delta) → new plan (e.g. waive one future instalment;
                                              re-spread the outstanding balance over k instalments;
                                              raise every target by 10 %)
  Once-only: yes / no   (a concession granted once per plan vs a policy that re-fires every pass)
```

**Rules:**
- Conditions are asked of the **delta**, never of the plan alone or of reality alone. Different businesses ask different questions (counts, quantities, ratios) of the same delta.
- A modifier is a **pure function** of the current plan and the delta. It never edits in place; it returns a new plan. This is what makes simulation free (below).
- Rules are **ordered**. Several may fire from one delta; each later modifier sees the plan the earlier one produced and the *same original delta*. Because the delta may then be stale relative to the plan (indices shifted, lines removed), a modifier must address lines by identity or by date, never by position.
- "Once only" is a **domain concept** ("the waiver is granted once per loan"), not a technical guard. Its scope is the plan's *lineage* (the loan, the product line across runs), not one version: record which once-only rules have fired with the lineage, so a successor plan cannot re-fire them.
- Re-planning has two shapes: **amend** the current plan (a new version of the same obligation) or **seed** the successor period's plan from this period's delta (next month's targets from this month's shortfall). Say which; the invariants and the settled-lines decision below apply to the first shape, the lineage-scoped latch to both.
- Every modifier states its **invariant**. Re-spreading the outstanding balance changes *how many* instalments and *when*, never *how much*: the remainder goes on the last instalment so the total is preserved exactly. Waiving an instalment reduces the total and says so.

**Design decisions to state explicitly:**
- **Absence**: conditions of the form "at least N things happened" never fire when nothing happened. If the business must react to *no payment at all*, add a condition over the missing bucket.
- **Settled lines**: already-satisfied planned items stay in the plan so its total still means "the whole obligation". Convention: a modifier may touch only lines not yet satisfied at the time of the delta; name the alternative if the domain wants otherwise (and note that removing a line that history refers to turns its payment into an "unexpected" item on the next pass).
- **Automatic or advisory**: does the rule change the plan, or propose a change for a human to confirm? The further execution has progressed, the costlier a plan change is — cheap while nothing has happened, a compensating process once it has — so a modifier must be told which lines execution has already reached.

**Simulation** needs no flag, no mode and no copy: because plans are values and modifiers are pure, a what-if is *"compute the delta and the rules again with a different plan, different actuals or a different tolerance"*. N plans × M executions × K tolerances are N×M×K deltas, none of which touches reality. A modifier invoked on demand ("try a 15 % buffer") has no delta condition: list it under Simulation with its parameters, not in the rule table. If scenarios must be kept and named, that is a Plan Version with a *scenario* purpose (Step 9) — one field borrowed from level 7, not the whole level.

---

### Step 9: Define Plan History and Integration Boundaries

**Plan history (level 7).** The reference behaviour at lower levels is a single *active* plan that re-planning replaces; the previous plan survives only if someone kept it. This is a design decision:

| Option | Keeps | Cannot answer |
|--------|-------|---------------|
| Single active plan | the current plan and the delta that produced it | "what was the plan before?", "compare execution against the original" |
| Plan Versions | every plan with *effective-from*, *reason* (which rule fired, which delta, or which human decision), *author*, and a purpose (*active*, *superseded*, *scenario*) | — |

At level 7 choose Plan Versions. State that "modified by / modified at" alone is an audit of *state*, not of *intention*, and does not meet the requirement.

**Boundaries.** Name what this model does and does not own:

| Concern | Owner | Relationship |
|---------|-------|--------------|
| Agreed terms, who ordered, who executes | ordering | upstream — supplies the planned items |
| Repayment plan templates, product parameters | product / pricing | upstream — defines how a plan is generated |
| Step ordering, parallelism | scheduling graph | upstream — produces the plan's sequence |
| Resource availability, reservations | inventory | sideways — the plan may reference reservations; it never allocates |
| Balances, penalties, waivers, interest | accounting | downstream — the delta and the re-planning are *facts* it may post |
| Notifications, dashboards | reporting | downstream — reads the delta and statistics |

**Rule:** this model computes the delta and applies plan changes. It does not decide *whether a payment was authorised*, *what the penalty costs*, or *whether the customer may be contacted*. Those live in the modules above and below.

---

### Step 9.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision in the draft model and verify each has a source:
- **(R)** — explicitly stated in requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred by fitting the model to sample numbers given in the requirements (e.g. a rounding rule that reproduces a known total)
- **(X)** — neither: assumed silently

**Decision checklist:**

| Decision area | Example decisions to check |
|---------------|---------------------------|
| Complexity level | Which of the 7 levels? Which layers are deliberately skipped? |
| Plan | Identity key? Amount? Date? Ordering invariant? Provenance stored rather than recomputed? |
| What counts as done | Raw reading or formula? Which one? |
| Dates, corrections, duplicates | Business date drives lateness? Superseding event recomputes the delta? Events have identity? |
| Tolerance | Shape(s)? Symmetric? Zero-baseline case? Composition? |
| Identity gate | Key must match before tolerance is consulted? |
| Matching policy | First fit or best fit? Group cap? One actual bound at most once? |
| Delta semantics | "Perfect" definition? Gross buckets with a net figure? |
| Statistics | Which counts/totals? Facts vs matches? On-time / late definition? |
| Resolution bridge | Direction? Projection rule? Calendar-aware? Partial period? Aggregate or item-level tolerance? |
| Re-planning | Which conditions? Which modifiers and their invariants? Order? Once-only? Automatic or advisory? Reaction to absence? Settled lines untouched? |
| History | Single active plan or versions with reason and effective-from? Named scenarios? |
| Boundaries | What is posted to accounting? What comes from ordering / product? |

**For every (X) decision found:**

1. If the decision has low impact (purely technical, easily changed): mark it as an explicit assumption in Implementation Notes.
2. If the decision affects business behaviour (tolerance symmetry, what counts as done, bridge, once-only, settled lines, history): **stop and ask** using `AskUserQuestion` before delivering the model — or, when it is unavailable, deliver with the assumption marked prominently.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

```markdown
# Plan vs Execution Archetype Model: [Domain Name]

## Complexity Level
[Level N — name. One sentence on why. Layers deliberately not modeled.]

## Concept Mapping

| Domain Concept | Plan vs Execution Archetype | Notes |
|----------------|-----------------------------|-------|
| ...            | ...                         | ...   |

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
[Every decision from the Step 9.5 checklist, not only the questions asked]

| Question | Answer | Source (R/A/I/X) |
|----------|--------|------------------|

## Plan Side

| Aspect | Decision |
|--------|----------|
| Planned item dimensions | [identity key / amount / date] |
| Ordering invariant | [yes, by … / none] |
| Plan identity | [per contract / per month / …] |
| Provenance | [stored result of … ; inputs referenced] |

## Execution Side

| Aspect | Decision |
|--------|----------|
| Execution event | [id, business date, recording date, source] |
| Actual item | [projection / same as event] |
| What counts as done | [formula; alternative rejected] |
| Lateness date | [business date] |
| Corrections / duplicates | [supersede & recompute; events have identity] |

## Comparison

### Tolerance
| Name | Shape | Parameters | Symmetric? | Reason text |
|------|-------|-----------|------------|-------------|

### Matching policy
[identity gate; first fit / best fit; group cap; actual bound at most once]

### Delta
| Bucket | Meaning in this domain | Amount counted |
|--------|------------------------|----------------|
| matched | ... | residual kept; added to the shortfall total: yes / no |
| missing | ... | full planned amount |
| unexpected | ... | full actual amount |
[Perfect = …; gross buckets + net figure]

### Statistics
| Statistic | Definition |
|-----------|------------|

## Resolution Bridge
[If level < 5, replace the table with the single line: Not applicable — plan and execution share a grain]
| Aspect | Decision |
|--------|----------|
| Direction | ... |
| Projection rule | ... |
| Partial period | ... |
| Tolerance level | item / aggregate |

## Re-planning Rules & Simulation
[If level < 6, replace the table with the single line: Not applicable — delta is reported, plan not changed by the system]
| Rule | Condition (on the delta) | Modifier | Invariant | Once-only? | Order |
|------|--------------------------|----------|-----------|------------|-------|
[Amend or seed shape; reaction to absence; settled lines; automatic vs advisory; simulation = recompute with other inputs; on-demand modifiers with their parameters]

## Plan History & Boundaries
[Single active plan / Plan Versions with reason, effective-from, purpose]
| Concern | Owner | Relationship |
|---------|-------|--------------|

## Worked Example
[One plan, one set of execution events, the delta with numbers, the rules that fired, the resulting plan]

## Implementation Notes
[Key decisions, assumptions for unanswered questions, edge cases, variable policies passed in]
```

---

## Common Patterns & Pitfalls

### Pattern: Three Things, Never Fewer

Plan, execution and delta are three separate values. Collapse any two and you lose a question: plan+execution in one record cannot answer "what if"; plan+delta in one record rewrites the delta whenever someone moves the goalposts. The delta being a third object is what makes N plans × M executions possible.

### Pattern: Plans Are Values, So Simulation Is Free

Recording execution creates a new fact; re-planning creates a new plan; nothing is overwritten. A what-if is the same computation with different inputs. If you find yourself adding an `is_simulation` flag, a defensive copy, or a "read-only mode", a value has become an entity somewhere.

### Pitfall: One Flat Status for Two Dimensions

`late / early / under / over / deviation` in a single enum cannot express "late and over" and always has unreachable values. Timing and amount are separate tolerances and separate statistics.

### Pitfall: Audit Without Intention

"Modified by Jane at 10:42" records that the plan changed, not why, from when, or what it was before. If the business asks those questions, that is level 7 and needs Plan Versions.

---

## Quality Checks

Before returning the model, verify (at level 1 only the first two apply; the rest presuppose level ≥ 2):

- [ ] Complexity level is stated and every skipped layer is marked *deliberately not modeled*
- [ ] Concept mapping table is present and complete; Unmapped Concepts section is present (even if empty)
- [ ] The plan is stored, addressable and immutable; provenance of planned numbers is named
- [ ] Execution events have identity, a business date and a recording date; "what counts as done" is a stated formula
- [ ] Every tolerance has a shape, parameters, a symmetry decision and a reason text
- [ ] Matching policy states the identity gate, first-fit or best-fit, and that an actual item binds at most once
- [ ] Delta has all three buckets, a "perfect" definition, a residual-counting choice and gross buckets with a net figure — or, if an aggregate-level tolerance was chosen, that choice stands in place of the bucket table
- [ ] Statistics distinguish identity (tolerance) from quality (on-time, short, over)
- [ ] Level ≥ 5: resolution bridge direction, projection rule and partial-period policy are stated and recorded with the delta
- [ ] Level ≥ 6: every rule has a condition on the delta, a modifier with an invariant, an order and a once-only decision; reaction to absence and settled-line policy are stated
- [ ] Level 7: Plan Versions carry reason, effective-from and purpose
- [ ] Boundaries name what is received from ordering / product and what is posted to accounting
- [ ] Worked example numbers are hand-checked and consistent with the tables
- [ ] All clarifying answers or assumptions are reflected in the model

---

## Example

**Input:** "A loan is repaid in five instalments of 100 PLN due on the 15th of January to May 2024. A payment counts as the instalment if it is within 5 groszy of the amount and no more than 10 days from the due date; a payment after the due date is late. If the borrower pays two instalments on time, one future instalment is waived. If any instalment is paid late, the outstanding balance is re-spread over three instalments across the same period. Each concession is granted once per loan. The bank must be able to show the original schedule after any change."

**Output:**

```markdown
# Plan vs Execution Archetype Model: Loan Repayment Schedule

## Complexity Level
Level 7 — plan history and intent. Re-planning rules (6), tolerance and delta (3), and a plan
with an ordering invariant (2) are all required; "show the original schedule" demands versions.
Level 4 (partial payments) and level 5 (resolution bridge) are deliberately not modeled — the
requirements mention neither; the tolerance still accepts a group so partials can be added later.

## Concept Mapping

| Domain Concept | Plan vs Execution Archetype | Notes |
|----------------|-----------------------------|-------|
| Repayment schedule | Plan | ordered by due date; one per loan |
| Instalment | Planned Item | amount + due date; no item identity beyond position |
| Payment received | Execution Event | own id, business date (payment date), recording date |
| "Counts as the instalment" | Tolerance | amount band AND date window |
| Late / on time | Statistic | quality verdict, separate from the tolerance |
| Waive an instalment | Re-planning Rule (modifier) | once-only |
| Re-spread the balance | Re-planning Rule (modifier) | once-only; total preserved |
| Original schedule | Plan Version | superseded versions retained with reason |

## Unmapped Concepts
None identified.

## Clarifying Questions & Answers

| Question | Answer | Source (R/A/I/X) |
|----------|--------|------------------|
| Is the date window symmetric (early payments also within 10 days)? | Yes; early is never late | (R) — the requirement says a payment *after* the due date is late, so early is on time |
| Which instalment is waived? | The next unpaid one | (X) — assumed; confirm |
| Do settled instalments stay in the schedule after re-spreading? | Yes; total still means the whole obligation | (X) — assumed |
| Is re-planning automatic? | Yes | (R) |
| In which order do the two rules apply? | Reward first, then restructure | (X) — assumed; waiving a line before re-spreading shrinks the balance that is then re-spread |
| First fit or best fit when several payments could satisfy an instalment? | First fit in due-date order | (X) — assumed; low impact at group cap 1 |

## Plan Side

| Aspect | Decision |
|--------|----------|
| Planned item dimensions | amount (PLN), due date |
| Ordering invariant | ascending by due date |
| Plan identity | one schedule per loan; versions per change |
| Provenance | generated from the loan product's repayment template; stored, not recomputed |

## Execution Side

| Aspect | Decision |
|--------|----------|
| Execution event | payment id, amount, payment date (business), booked-at (recording) |
| Actual item | same shape as a planned item: amount + date |
| What counts as done | amount received; refunds supersede with a new event |
| Lateness date | payment date |
| Corrections / duplicates | a reversal is a new event; delta recomputed; events have identity |

## Comparison

### Tolerance
| Name | Shape | Parameters | Symmetric? | Reason text |
|------|-------|-----------|------------|-------------|
| instalment_match | absolute amount band AND date window | 0.05 PLN; 10 days | yes (early accepted) | "amount within 0.05 and date within 10 days" |

### Matching policy
No identity gate. First fit in due-date order; an actual payment binds at most once. Group cap 1 —
the tolerance is collection-shaped, so raising the cap and adding an "aggregate before the due date"
shape later adds partial payments without changing the delta structure.

### Delta
| Bucket | Meaning in this domain | Amount counted |
|--------|------------------------|----------------|
| matched | instalment settled (possibly late, possibly a few groszy off) | residual kept |
| missing | instalment not settled | full instalment amount |
| unexpected | payment no instalment wanted | full payment amount |
Perfect = no leftovers (residuals within tolerance are fine). Gross buckets plus net difference.

### Statistics
| Statistic | Definition |
|-----------|------------|
| on_time | matches whose payment date ≤ due date |
| late | matches whose payment date > due date |
| missing_amount | Σ missing instalments; accepted residuals not added (leftovers-only reading) |
| net_difference | Σ actual − Σ planned |

## Resolution Bridge
Not applicable — plan and execution share a grain (per payment).

## Re-planning Rules & Simulation
| Rule | Condition (on the delta) | Modifier | Invariant | Once-only? | Order |
|------|--------------------------|----------|-----------|------------|-------|
| reward | on_time ≥ 2 | waive the next unsettled instalment | total decreases by that instalment | yes | 1 |
| restructure | late ≥ 1 | re-spread outstanding balance over 3 instalments between first and last outstanding due dates; remainder on the last | total unchanged | yes | 2 |
Absence: neither rule fires when no payment arrives; missing instalments are reported only.
Settled lines stay in the plan and are never modified. Rules are automatic. Simulation = recompute
with another tolerance or another schedule version; nothing is written.

## Plan History & Boundaries
Plan Versions: each re-planning creates a new version with effective-from = analysis date, reason =
rule name + delta id, purpose = active; the previous version becomes superseded.
| Concern | Owner | Relationship |
|---------|-------|--------------|
| Loan terms, repayment template | product / ordering | upstream |
| Balance owed, interest, penalties | accounting | downstream — delta and versions are posted as facts |

## Worked Example
Plan v1: 100.00 due 15 Jan, 15 Feb, 15 Mar, 15 Apr, 15 May 2024 — total 500.00.
Execution: one payment, 100.00 on 20 Jan 2024.

Delta D1 (v1, tolerance instalment_match):
- matched: 15 Jan ← 20 Jan (amount diff 0.00 ≤ 0.05; 5 days ≤ 10) — reason recorded; late
- missing: 15 Feb, 15 Mar, 15 Apr, 15 May — 400.00
- unexpected: none
- statistics: on_time 0, late 1, missing_amount 400.00, net_difference −400.00

Rules: reward (on_time ≥ 2) does not fire. restructure (late ≥ 1) fires, once-only latch set.
Outstanding = 400.00 over 15 Feb → 15 May (90 days; interval 45 days).
Split 400.00 into 3: 133.33, 133.33, 133.34 (to the minor unit; remainder on the last — the stated invariant) — sums to 400.00.
Dates: 15 Feb, 31 Mar, 15 May.

Plan v2 (effective 20 Jan 2024, reason "restructure fired on D1"):
100.00 on 15 Jan (settled) · 133.33 on 15 Feb · 133.33 on 31 Mar · 133.34 on 15 May — total 500.00.
v1 is retained as superseded; the bank can show it and can recompute D1 against it at any time.
Next pass compares new payments against v2; restructure cannot fire again on this loan.

## Implementation Notes
- Tolerance is a parameter of the comparison; the 0.05 PLN / 10-day values come from the loan
  product and are passed in, not hard-coded.
- Assumption (X): "one future instalment is waived" means the next unsettled one — confirm.
- Assumption (X): settled instalments stay in the schedule; the total means the whole obligation.
- Assumption (X): rule order is reward-then-restructure. Reversing it re-spreads the full outstanding
  balance and only then waives a line, giving a different schedule from the same delta — confirm.
- Partial payments deliberately not modeled; the tolerance already accepts a group, so adding an
  "aggregate before due date" shape later does not change the delta structure.
- Recording date is kept on every payment event for reconciliation and never used for lateness.
```

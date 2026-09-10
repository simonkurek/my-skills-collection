---
name: archetype-skill-builder
description: Create or extract a `<name>-archetype-mapper` skill from a software archetype's sources (reference implementation, training transcript, notes) using a sibling mapper as the structural template. Distills sources with parallel agents, keeps the skill at architecture/DDD/business level, verifies it with a fidelity review and a dry run, and registers it in the archetype scanner.
argument-hint: "[archetype name] [paths to sources: code module, transcript/docs, template skill]"
---

# Archetype Skill Builder

Turn the raw material about a software archetype — a reference implementation, a training transcript, notes, an existing sibling skill — into a new `<name>-archetype-mapper` skill that a modeler can apply to any domain.

**Output goal**: A `SKILL.md` in the skills collection that (1) matches the structure and tone of the existing mapper skills, (2) describes the archetype as a way of thinking at the architecture / DDD / business level, not as one codebase's API, (3) lets a domain adopt only the part of the archetype it needs, and (4) has been verified against its sources and test-driven on an independent domain.

## When to Use

- A new archetype has a reference module and/or a transcript and no mapper skill yet (party, product/catalog, ordering, inventory, rules, plan-vs-execution, graphs…)
- An existing mapper skill drifted toward implementation detail and needs to be re-extracted
- The `archetype-scanner` registry references a `<id>-archetype-mapper` that does not exist

## When NOT to Use

- The material is a single design pattern, not an archetype (no fit test, no domain mapping to produce) — write a normal skill instead
- The sources are only code with no explanatory material: the skill would inherit implementation choices as if they were the archetype. Gather at least one narrative source first (docs, transcript, book chapter, author's talk).

---

## Guiding Principles

These come from building `pricing-archetype-mapper` and from the corrections made along the way. Apply them throughout.

1. **The reference code is one example, not the archetype.** Never carry class names, enum values, factory methods, operator names or runtime quirks into the skill. Carry the *concepts* they implement.
2. **Architecture / DDD / business altitude.** A reader should be able to apply the skill in any language or stack. Name concepts (Calculator, Component, Version, Validity, Applicability…), describe their responsibilities and relationships, and stop there.
3. **Not every domain needs the full archetype.** Put a complexity ladder (or an equivalent fit gradient) early and let it gate later steps. "Deliberately not modeled" is a valid outcome.
4. **Where sources disagree, present a decision, not a rule.** The transcript and the code will differ in places. The skill names both options and asks the modeler to choose.
5. **Where the trainers leave something open, keep it open.** Do not promote a "we chose X in this course" into "you must do X".
6. **Match the sibling skills.** Same section order, same clarifying-question convention (`AskUserQuestion`, "To zależy / It depends" as last option), same (R)/(A)/(I)/(X) sanity check, same output-format-then-quality-checks-then-example ending.
7. **Clarity beats completeness.** Target the length of the shortest good sibling (roughly 500–700 lines). Every rule must earn its place.
8. **The example must be internally consistent.** Its arithmetic is checked by hand, its tables agree with its tree, and it never contradicts its own input.

---

## Workflow

### Step 0: Gather Inputs

Collect, or ask for, the following. Do not start distilling until all are located.

| Input | Where to look | Notes |
|-------|---------------|-------|
| Archetype id and name | user, scanner registry | id is kebab-case; skill name is `<id>-archetype-mapper` |
| Reference implementation | `archetypes/<module>/src/main`, `src/test` | tests are the best source of worked examples |
| Narrative sources | `archetypes/docs/<module>.md` (training transcript, often Polish), README, book chapters, slides | at least one is required |
| Template skill | the sibling mapper the user points to, else the most recently reviewed one | copy its **structure**, not its content |
| Target directory | `plugins/archetype-modeling/skills/<id>-archetype-mapper/SKILL.md` | the collection is a git repo published as a plugin marketplace; commit the new skill and bump the plugin version |
| Scanner registry | `plugins/archetype-modeling/skills/archetype-scanner/SKILL.md` | one row per archetype with a one-line fit question |

Read the template skill fully yourself. Read only the *shape* of the sources (file list, sizes, headings) — the distillation is delegated.

---

### Step 1: Distill Sources in Parallel

Launch one agent per source, in a single message, writing to the scratchpad. Using different models for different sources is fine and gives a second perspective. Each agent gets the same framing: *"You are preparing source material for a Claude Code skill called `<id>-archetype-mapper`."*

**Narrative distiller** (transcript / docs) — ask for, in English:
1. Lesson or chapter index with 2–4 sentence summaries
2. Core concepts: definition, why it matters, domain examples used, every design rule / heuristic / anti-pattern stated (quote or closely paraphrase, with lesson reference)
3. Fit test material: positive signals, negative signals, borderline cases and how the authors resolve them
4. Every decision the authors surface, phrased as a question a skill would ask
5. Worked examples reproduced compactly with numbers
6. Exercises and their intended solution direction
7. Glossary (source language → English)
8. Anything absent from the source that a reader might expect — marked *absent*, never invented; uncertainties marked `[unclear]`

**Code distiller** (reference module + tests) — ask for, in English:
1. Building-blocks catalog: each public concept, its responsibility, invariants, relationships; a text diagram
2. The concrete variants of each concept (calculator kinds, account types, constraint operators…) with semantics
3. Composition / aggregation semantics, ordering rules
4. Time model: validity, versions, selection rule, what happens in gaps
5. Conditional behaviour: what can be constrained, what happens when nothing applies
6. Every scenario test as a business story with the computed numbers
7. Design rules and *limitations* inferred from code — clearly separated from archetype-level rules
8. Vocabulary the code uses (kept for the reconciliation step, not for the skill)

Wait for both. Read the outputs in chunks; they will be large.

---

### Step 2: Reconcile and Set the Altitude

Before writing anything, build two short tables in the scratchpad.

**Keep / drop table** — for every item in the distillations:

| Item | Verdict | Reason |
|------|---------|--------|
| concept with responsibilities (e.g. "component owns dependencies between children") | keep | archetype-level |
| generic shape (e.g. "step function", "percentage of a base") | keep, rename generically | archetype-level |
| enum value, class name, method name, operator name | drop | implementation |
| runtime quirk (fallbacks, hard-coded currency, adapter cost, string-only context) | drop | implementation; at most one line in Implementation Notes guidance |
| rule the code enforces that the narrative does not mention | keep only if it is a genuine modeling constraint (e.g. acyclic dependencies); else drop | |

**Disagreement table** — every place the narrative and the code differ (e.g. composite applicability OR-over-children vs. own constraint):

| Topic | Narrative says | Code does | In the skill |
|-------|---------------|-----------|--------------|
| … | … | … | present as a modeler decision with both options |

Also decide the **complexity gradient**: if the narrative has a maturity ladder, use it verbatim as Step 1 of the skill and map each level to the skill steps it unlocks. If not, derive one from the concepts (each layer of the archetype = one level).

---

### Step 3: Design the Skill's Spine

Write, in a few lines each, before drafting:

- **Core fit question** — one sentence a modeler can ask of any domain. Add the *counter-questions* that route to sibling archetypes ("how much does S have?" → accounting; "what state?" → state machine).
- **Signal table** — 8–12 rows, half positive, half negative, one or two borderline. Negative rows name the archetype that owns the concern.
- **Concept vocabulary** — the 6–10 generic nouns the skill will use, with one-line responsibilities. This is the skill's language; never introduce a code identifier later.
- **Step list** — one step per concept layer, in the order a modeler naturally works (value → parameters → math → semantics → time → conditions → boundaries), plus the fixed Steps 0, 2 (questions), 3 (concept mapping) and 9.5 (sanity check).
- **Clarifying questions** — Category A (standard decisions, ≤ 8) and Category B (gap triggers, ≤ 10), taken from the narrative's surfaced decisions.
- **Example domain** — prefer the narrative's flagship example; verify its numbers against the reference tests if they exist.

---

### Step 4: Write the SKILL.md

Follow the template skill's section order exactly:

```
frontmatter (name, description, argument-hint)
# Title + one-paragraph framing + Output goal
## When to Use
## When NOT to Use — Fit Test   (core question, signal table, borderline cases, not-fit output block)
## Mapping Workflow
   Step 0 Get Requirements
   Step 1 Assess Complexity Level   (with "which steps to run" guidance)
   Step 2 Ask Clarifying Questions  (Category A, Category B, "It depends" handling, no-AskUserQuestion fallback)
   Step 3 Map Domain Concepts        (mapping table + Unmapped Concepts, always present)
   Steps 4…N  one per concept layer
   Step N.5 Decision Sanity Check   ((R)/(A)/(I)/(X), checklist table, stop-and-ask rule)
## Output Format   (markdown skeleton; sections match the steps one-to-one)
## Common Patterns & Pitfalls
## Quality Checks
## Example   (input → detected level → full output in the Output Format)
```

**Writing rules:**
- Concept tables have columns *Concept / Meaning / Key rules*, never *Class / Method*.
- Variants are described by shape and formula (`f(q) = base + ⌊q/step⌋ × increment`), not by type name.
- Policies are described by intent ("no overlaps / overlay / anything"), not by enum.
- Every "Rule:" must trace to the narrative or to a structural necessity. Anything else is "Convention:" with the alternative named.
- Disagreements from Step 2 appear as *"this is a design decision: …"* with both options.
- The output format has an "omit if level < N — say so" note on layers that lower levels skip.
- Quotes from the narrative are welcome when they carry a rule memorably; keep them short.
- Write the example last, compute every number by hand, and make sure tables, tree and notes agree.

---

### Step 5: Verify with Parallel Agents

Launch both in one message; neither may edit the skill.

**Fidelity reviewer** (a strong model, fresh context): reads the skill, the two distillations and, where doubtful, the raw sources. Checks factual errors, example arithmetic, contradictions with the narrative, material omissions, internal consistency (steps ↔ output format ↔ quality checks ↔ example), and usability. Reports Critical / Should fix / Nice to have with line numbers and proposed replacement text.

**Dry-run tester** (a different model): applies the skill literally to an independent domain — ideally the narrative's exercise or homework, with its expected numbers — without `AskUserQuestion`. Produces the full model plus a friction report: ambiguities, steps whose output is never consumed, tables that could not be filled, missing fallbacks, and what worked well.

Then run an **altitude check** yourself:

```
grep -c '<any code identifier from the code distiller's vocabulary list>' SKILL.md
```

must be zero for enum values, class names and method names.

---

### Step 6: Apply Fixes and Re-check

- Apply Critical and Should-fix items and the actionable friction items in one scripted pass, asserting each replacement matches exactly once.
- Re-verify the example arithmetic after any change to it.
- Re-read the section outline (`grep -n '^## \|^### Step'`) to confirm numbering and cross-references.
- If fixes pushed the file well past the template's length, cut: merge tables, drop quotes, move implementation caveats out.

---

### Step 7: Register and Report

- Add or confirm the row in `archetype-scanner`'s registry: `| <id> | <id>-archetype-mapper | "<one-line fit question>" |`.
- Do not commit unless the collection is a git repo and the user asked.
- Report: what was built, how it was verified (agents, models, what they caught), the numbers the example reproduces, and what was deliberately left out.

---

## Output Format

The deliverable is the new `SKILL.md`. Accompany it with a short build note in the reply (not a file):

```markdown
## <id>-archetype-mapper — build note

**Sources**: [code module], [narrative], [template skill]
**Complexity gradient**: [ladder used / derived]
**Disagreements presented as decisions**: [list]
**Dropped as implementation detail**: [short list]
**Verification**: fidelity review ([model]) — [n critical, n should-fix, all applied]; dry run ([model]) on [domain] — [numbers reproduced]
**Registered in scanner**: yes / already present
**Left out**: [anything deliberately not modeled]
```

---

## Quality Checks

Before returning, verify:

- [ ] Skill name is `<id>-archetype-mapper` and matches the scanner registry
- [ ] Section order matches the template skill; Output Format sections match the steps one-to-one
- [ ] Complexity level (or equivalent) is Step 1 and states which later steps each level unlocks
- [ ] No class, enum, factory, method or operator names from the reference code appear anywhere
- [ ] Every "Rule" traces to the narrative or a structural necessity; conventions name their alternative
- [ ] Every narrative-vs-code disagreement is presented as a decision with both options
- [ ] Fit test has a core question, counter-questions routing to sibling archetypes, and a not-fit output block
- [ ] Clarifying questions include the "It depends" option and a no-`AskUserQuestion` fallback
- [ ] Sanity check uses (R)/(A)/(I)/(X)
- [ ] Example: numbers hand-verified, tree ↔ tables ↔ notes consistent, never contradicts its own input
- [ ] Fidelity review and dry run were run by separate agents and their findings applied
- [ ] Length is within ~20% of the template skill

---

## Pitfalls Seen in Practice

- **Over-indexing on the reference code.** The first pricing draft listed enum values, adapter costs and a hard-coded currency bug. The user's verdict: *"too much on specific implementation in the given project, which is only an example."* Distill code for concepts and worked numbers, then leave it behind.
- **Example that agrees only at the sampled point.** A step function reproduced the tariff at 12 kWh but not at 16. Check the whole domain of every function in the example.
- **Contradiction inside the example.** One table versioned the root composite, another versioned the child, and only one of them kept VAT correct. Build the example tree once and derive every table from it.
- **Stating an open question as a rule.** "Definition parameters live on the calculator" was the course's choice, not the archetype's. The reviewer caught it because the narrative said "it depends".
- **Silently adopting the code's semantics where the trainers taught differently.** Composite applicability was one rule in the code and the opposite in the transcript. Present both.
- **Broken cross-references after renumbering.** Re-grep step numbers after every structural edit.
- **Length creep from review fixes.** Each fix added lines; the result was 35% longer than the sibling and harder to read. Budget for a cutting pass.
- **Missing fallbacks.** The dry-run agent had no `AskUserQuestion`; the skill had no instruction for that case. Every interactive step needs a non-interactive path.

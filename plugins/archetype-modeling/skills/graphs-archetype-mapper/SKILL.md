---
name: graphs-archetype-mapper
description: Transform domain requirements into a Graphs Archetype model. Detects the hidden network behind if-cascades and flag soups, names nodes, edges and their semantics, picks the business question and the structure that answers it (cycle, path, zone, order, parallel class, connector), separates topology from policy graphs, and sets the deployment boundary. Produces an implementable model with explicit concept mapping, unmapped concepts and boundaries.
argument-hint: "[domain requirements or feature description]"
---

# Graphs Archetype Mapper

Transform any domain description in which **relations between things determine what the system does** into an explicit graph model. The trainers' thesis: *"your system pretends to be a line, but it is often a network"*. The network already exists, hidden in ifs, loops, flags, recursion and snapshots. This skill makes it explicit and lets a modeler ask the graph the questions the business is really asking.

This is not an algorithms skill. Every mechanism below (cycle detection, path enumeration, connected components, topological ordering, colouring, articulation points, intersection and union) is a solved, off-the-shelf problem: a single library call, or a few dozen lines written once (Step 9). The modeling work is deciding **what the nodes are, what an edge means, which question you ask, and which semantics live in which graph**.

**Output goal**: A complete, implementable model that names each graph's nodes, edges and semantics, the business questions and the structures that answer them, the separation between what is possible and what is allowed, and the boundary at which the graph lives.

## When to Use

**Use this skill when:**
- Dependencies between entities decide flows, blocks, limits, retries or execution order
- Individually impossible requests could succeed together (swaps, netting, matching rings)
- Someone passes through stages with detours, shortcuts and blocks, and asks "how do I reach X" or "what is the fastest way"
- A change in one place drags in parties that were never directly involved
- The business asks "in what order", "what can run in parallel", "whom will this affect", "what holds everything together"
- Swapping two rules changes the result, and every new rule inserts itself *between* the others

**Output is useful for:**
- Domain modeling sessions before implementation
- Replacing brute-force simulation and flag-based rule chains with an additive model

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the graphs archetype does not fit, and briefly explain why.

### The core question

> *"Does the outcome depend more on the arrangement of connections between things than on a list of steps?"*

If **yes** → graphs archetype likely fits.
If the natural question is **"how much X does S have?"** → accounting. If it is **"what does X cost in this context?"** → pricing. If it is **"who plays which role towards whom?"** → party. If it is **"what was submitted and agreed?"** → ordering. If it is **"how far is what happened from what was planned?"** → plan-vs-execution. Do not map.

Second gate, from the trainers: *"If you have three business cases and each one is different, you probably do not need graph theory. You simply have a set of exceptions, and that is okay."* A stable, non-growing sum of cases is level 0. Do not stop here: run Steps 0–3 so the concept mapping shows what would have become nodes and edges, then recommend plain conditionals and stop (Step 1 guidance).

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "none of these requests can execute alone, but together they could" | ✅ Yes — cycle |
| "it depends how the customer got here"; "fastest way to get the discount"; "is X reachable at all" | ✅ Yes — path / reachability |
| "if we accept this, four teams have to coordinate instead of one" | ✅ Yes — influence zone |
| "this must finish before that"; "these two can run in parallel, the third only after" | ✅ Yes — ordering + parallelism |
| "everything flows through X"; "without X nothing happens" | ✅ Yes — connector |
| "we keep adding `status` / `enabled` / `isAllowed` / `limit` fields to the relation" | ✅ Yes — a second (policy) graph is being born |
| "swap two ifs and the test result changes" | ✅ Yes — logic rests on structure, not sequence |
| "user earns / spends N units; balance cannot go below zero" | ❌ No — accounting |
| "ticket goes open → assigned → resolved", one path, no branching | ❌ No — plain state machine; a journey needs alternative routes |
| "three different cases, stable, will not grow" | ❌ No — write the ifs |
| "recommend items to millions of users from a giant graph" | ❌ Out of scope — an obvious graph domain, not a hidden one |
| "order has a parent order and child orders" (a foreign key, nothing computed over it) | ⚠️ Borderline — does the relation *change what happens*? |
| "slot is free / booked" | ⚠️ Borderline — a quantity (accounting) unless owners want to *swap* |

### Borderline cases — how to decide

Use the trainers' three recognition heuristics. All three are heuristics, not rules.

- **Relationship analysis.** A relation modeled as a foreign key only is fine *until it carries semantics*: it influences the logic of some action, its meaning changes over time, relations influence other relations, self-referential one-to-many and many-to-many links appear. Test: *are objects connected to be found, or to compute, synchronise, balance, reconcile?* The second is a graph. Papering over it with "a separate table, a document with a type, a validity date" hides the problem.
- **Bridge words.** Listen for the words that connect domain language to graph language. Obvious: *cycle*. Synonyms: *loop, coupling, feedback*. Disguised: *dependency, block, flow, order, link, reachability, path, shortest route, netting, capacity, balance*. Ordering family: *before, after, in parallel, until, only when*. Influence family: *it depends who is working next door, don't run that in parallel, we have to synchronise the start*. Connector family: *everything flows through X, X coordinates everybody*. Path family: *it depends how they got here, rehabilitation path, route*. Frustration family, signalling two graphs: *not all relations are equal, just because it is connected doesn't mean it can work together*.
- **Draw it.** *"Take a sheet of paper and sketch who with whom, what with what, in which direction. One drawing tells you more than a thousand user stories, because the stories differ only in parameters, not in structure."* If the drawing shows a cycle, islands, a funnel or a single node holding groups together, you have a graph.

Two more tests decide the borderline rows:
- **Foreign key vs edge**: does the outcome of any operation depend on *who is connected to whom, in what order, whether somebody blocks somebody*? If yes → edge. If the link only locates a record → foreign key.
- **State machine vs journey**: a single mandatory sequence is a state machine and needs no graph algorithm. The archetype pays off when there are *alternative routes* to a state, the route taken matters, or the business asks "fastest / cheapest / reachable at all".

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The graphs archetype requires that the outcome depend on the arrangement of connections
between things, with a business question answered by a structure in that network
(cycle, path, zone, order, parallel class, connector). This domain is
[a ledger / a price computation / a party model / a stable set of cases / a single
sequence of states / ...] because:

- [specific reason from the requirements]
- The natural question is "[how much / what does it cost / who plays which role / ...]",
  not "what does the arrangement of connections allow?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — what things are connected, what does a connection mean, and what question does the business ask about the whole?"

---

### Step 1: Assess Complexity Level

Locate the **highest applicable level** in the requirements. Higher levels include all lower levels. Model **to that level, not past it**.

This ladder is a convention of this skill, not the trainers': the narrative supplies only the cost gate (level 0) and the deployment ladder (level 5). Use it to decide how far to model; if a domain does not sort cleanly onto it, describe the layers present and say so rather than forcing a number.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 0 | **Stable sum of cases** | Three or so cases, each different, no growth expected | (none — ifs are the right answer) |
| 1 | **One graph, one question** | Things connect; one business question about the whole (a cycle, a path, a zone, an order) | brute-force simulation with snapshots and "deficit" bookkeeping; if-cascades whose order matters |
| 2 | **Enriched relations** | Edges or nodes carry numbers or attributes; "fastest / cheapest / least risky"; "only above 25 dB"; "only when the shared supply is loaded" | weights hard-coded into the loop; thresholds as more ifs |
| 3 | **Two semantics** | What is *possible* differs from what is *allowed*; rules change faster than the structure; `enabled` / `isAllowed` / `limit` fields accrete on a relation; or a universal relation (physics, regulation) must be localised to places and merged with local quirks | one graph with typed, flagged, exception-ridden edges; "patchwork" of logic and permissions in one method |
| 4 | **Second questions** | Having ordered, the business asks what can run in parallel; having found zones, it asks which node splits them; having found a cycle, it asks which edges cause it | single-threaded execution of independent work; "it would be faster if not for X" left unexamined |
| 5 | **Graph as a shared capability** | Several contexts need the same network; the network has its own life cycle and rule-constrained global state | each context rebuilds a partial copy; or a "graph service" that speaks edges and nodes |

**Guidance — which steps to run:**
- Level 0: The archetype is overkill. Document the level, recommend plain conditionals, and stop after Step 3.
- Level 1: Core archetype — Nodes & Edges (Step 4) + Question & Structure (Step 5) + Boundaries (Step 9).
- Level 2: Add Weights & Predicates (Step 6).
- Level 3: Add Semantic Layers (Step 7).
- Level 4: Add Derived Questions (Step 8).
- Level 5: Expand Step 9 into a deployment decision.

Partial adoption is normal. Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps. Ask about **two categories** in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Always include **"To zależy / It depends"** as an explicit last option in every question.

#### Category A — Standard graph decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as design choices:

- **Node identity**: What is a node — the resource, the state, the stage, the party, the booking? Are there several candidate identities (a slot vs its owner, a process vs a process-in-a-place)?
- **Edge meaning**: What does a connection mean — a want, a condition, an influence, a precedence, a conflict, a transfer? Does the same connection mean different things to different stakeholders?
- **Direction**: Does the arrow carry meaning (cause, responsibility, "before")? Or does even one-sided influence force two-way coordination, so direction should be dropped for this question?
- **Cycle polarity**: Is a cycle the desired outcome (an executable group) or a defect (something that blocks ordering)?
- **Rule ownership**: Who decides which connections are permitted — the graph module, or its client? (The trainers' firm answer: the client.)
- **Runtime or persistent**: Is the graph rebuilt from entity data to answer a local question, or is it a model with its own life?
- **Unmatched events**: When an event arrives that matches no transition from the current state, is that a no-op or an error?
- **First or all**: When several cycles or paths exist, is the first one enough, or must all be found and ranked?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the graphs archetype supports but the requirements do not mention**. Reason freely; examples:

- **Weights**: Should edges or nodes carry time, cost, risk, strength, delay? On edges, on nodes, or both?
- **Thresholds**: Should a connection count only when a predicate holds (above a limit, within a period, for a pair of departments)?
- **Time overlap**: Do connections exist only between things that overlap in time (bookings in the same hour)?
- **Second graph**: Do the rules change at a different tempo than the structure? Is a permanent connection really an event in time?
- **Localisation**: If a universal relation (physics, regulation) must be applied to places or instances, does it apply everywhere, only between adjacent places, or only where explicitly declared?
- **Parallelism**: Once ordered, what may run at the same time? How many independent environments are needed?
- **Connectors**: Which node, if removed, splits the network — and is that a failure to guard against or a bottleneck to remove?
- **Counterfactuals and comparisons**: "What if event E had not happened?" "Which routes are shared between the standard and VIP programmes?"
- **Direct vs indirect reach**: Do stakeholders need only the count of direct contacts, or the whole domino reach?

Collect answers before proceeding. If the user cannot answer, document the assumption in **Implementation Notes**. If `AskUserQuestion` is unavailable (non-interactive run), list the questions in prose, pick a reasonable default, and mark it (X).

#### Handling "it depends / both / varies by situation" answers

If the user selects **"To zależy / It depends"**, the variability itself becomes part of the model:

- Variable *permission* ("depends on department, on limits, on the day") → it is a **policy graph**, built by the client and intersected with the topology (Step 7). Do not push the condition into the algorithm.
- Variable *strength* ("depends on the noise level") → it is a **weight with a predicate** on the edge (Step 6).
- Variable *direction* ("for money the arrow matters, for coordination it doesn't") → keep direction in the base graph and choose per question (Step 5).
- Variable *node identity* → model the graph at the finer identity and derive the coarser one by projection (Step 4).

Note in **Implementation Notes** which module supplies each variable input. Variability is data the graph consumes, not a branch inside it.

---

### Step 3: Map Domain Concepts to Graph Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Graph Archetype | Notes                          |
|----------------------|-----------------|--------------------------------|
| [domain noun/verb]   | Node / Edge / Direction / Weight / Predicate / Topology Graph / Policy Graph / Question / Structure (cycle, path, zone, order, parallel class, connector, feedback set) / Mechanism | [why] |
```

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear graph archetype equivalent:
- [concept] — [reason it doesn't fit / decision needed]
```

This section must be present even if empty (`None identified`). Expect the ledger (balances), the price, the parties' roles and the persistence of entities to land here — the graph reads them, it does not own them.

---

### Step 4: Identify Nodes and Edges

Name every graph the model needs. Most level-1 domains have one; level 3 has at least two over the same node space.

**For each graph, decide:**

| Decision | Options | Rule |
|----------|---------|------|
| **Node** | the contested resource, the state a subject is in, the stage of a process, the party, the booking (thing + place + time) | Pick the identity the *question* is about. A cycle of swaps is about resources; a permission to swap is about owners. If two graphs must be combined, their nodes must be the same kind of thing — re-project one onto the other's identity. |
| **Edge** | a want (from current to desired), a transition condition, an influence, a precedence ("before"), a conflict ("cannot coexist"), a transfer | One meaning per graph. *"The same edge may mean interdependence, order, blocking, synchronisation or priority — and when you change its meaning, the question the graph can answer changes too."* |
| **Direction** | directed, undirected | A domain decision, not a technical one. Keep arrows for cause, responsibility, money, information, "before". Drop them where one-sided influence forces two-way coordination. Ordering requires direction; zones ignore it. |
| **Multiplicity** | one edge per pair, or many | Several wants between the same two nodes, several dependency reasons between the same two stages — say whether they collapse or stay distinct. |
| **Lifetime** | built at runtime from entity data, or persisted | Default: runtime. *"The source data is in an entity database; what you extract from it is already a dependency graph."* Small, slow-changing inputs (physics, adjacency, stage rules) are reference data loaded at runtime, not a persisted graph. Persist only what has its own life cycle (Step 9). |
| **Time filter** | none, or edges only between things overlapping in time | Bookings in the same hour get a temporary edge; bookings in different hours do not. |
| **Built by** | the domain module, or its client | Topology is the module's; policy is the client's (Step 7). Name the module, not a layer label. |


**Rule:** filter the trivial cases before building the graph. A request for a free resource executes immediately and never becomes an edge.

**Decision — who evaluates a transition condition.** The narrative puts the rule on the edge (*"a condition can be any rule"*); the topology/policy split argues for keeping evaluation outside it. Two options: (a) the edge only **names** the condition, the caller decides it is fulfilled and reports the event — the mechanism stays rule-free and Step 7 stays possible; (b) the edge carries an evaluable condition the graph resolves itself — fewer round-trips, but rules leak into the mechanism. Record the choice explicitly and pair it with the unmatched-event decision (no-op or error).

---

### Step 5: Choose the Question and the Structure

Each business question maps to one structure in the graph and one ready mechanism. Pick the row(s) the requirements actually ask for. Do not add rows "because the graph can".

| Business question | Structure | Mechanism | Precondition | Level | On empty result |
|-------------------|-----------|-----------|--------------|-------|-----------------|
| "Which of these requests can execute *together* when none can alone?" | cycle | cycle detection | directed; one resource, one owner | 1 | no group this run; nothing changes |
| "What must happen for me to reach X? Is X reachable at all?" | path | all simple paths from the current state; reachability | directed | 1 | X is unreachable from here |
| "How do I reach X fastest / cheapest / with least risk?" | cheapest path | path enumeration ranked by a weight function | directed | 2 | as above |
| "What would have happened if event E had not occurred?" | counterfactual path | replay the journey with the edge for E removed or a different edge taken | history of events kept | 2 | the route without E does not exist |
| "Whom will this change affect? Who has to synchronise with whom?" | zone (connected component) | connected components, ignoring direction | — | 1 | the element is isolated |
| "How many parties must I talk to *directly*?" | degree | count of a node's edges | — | 1 | none |
| "Will accepting this merge groups that were independent, and is the enlarged group still acceptable?" | zone growth | connected components before and after adding the element; compare zone count and size | — | 1 | a new singleton zone; no merge |
| "In what order must these run?" | order | topological ordering | directed **and acyclic** | 1 | a cycle blocks ordering (see polarity) |
| "What can run in parallel? How many independent environments do we need?" | parallel classes | minimal vertex colouring over the *conflict* graph | conflict edges are undirected | 4 | one environment suffices |
| "Which single element holds everything together? What happens if it disappears?" | connector (articulation point) | articulation-point detection on an existing graph | a graph already built | 4 | no single point; nothing to guard or remove |
| "Which edges make this graph cyclic, and is removing them a loss or a clean-up?" | feedback set | feedback arc set | cycles are a defect here | 4 | already acyclic |
| "What works today, given both structure and rules?" | intersection | edge-wise AND of two graphs | same node identity | 3 | nothing is both possible and allowed |
| "What could work if the rules of one dimension were lifted?" | union | edge-wise OR of two graphs | same node identity | 3 | — |
| "Which routes hold in both programmes, and which are unique?" | shared paths | intersection of two journey graphs, then path enumeration | same node identity | 3 | the programmes share no route |

**Empty result is an answer.** State what it means per question, as in the last column. An empty result is never a partial application.

**Zone growth:** keep both levels of answer: degree for "how many parties do I negotiate with directly", component size for "how far the domino reaches". Adding an element may merge two zones into one; when it does, report the new count and size and let the caller decide whether the coordination cost outweighs the benefit. The graph reports the merge, it does not price it.

**Cycle polarity rule:** decide it explicitly. In a swap system the cycle *is* the executable group. In a precedence system a cycle blocks ordering and is a defect. This is a design decision when the graph must be acyclic: reject the offending edge the moment it is declared, or accept the graph and compute the feedback arc set to propose which edges to cut. The trainers name both; the modeler chooses.

**Cycle execution convention:** when a cycle of exchanges executes, release every "from" first, then assign every "to". No resource is held twice mid-swap. The alternative (pairwise swaps in sequence) needs intermediate states and is what the brute-force version had to snapshot.

**First vs all:** *"we look for the first cycle, though we could equally well find all of them."* If fairness, priority or scoring between competing groups matters, find all and rank; otherwise the first is enough. Anything outside the chosen cycle is left untouched, not partially applied.

**Additivity rule (journeys):** a new business case is a new edge — start state, condition, target state. *"You do not have to find a place in the chain of ifs, check priorities or watch the ordering."* Tests become topology tests: *does a path exist from A through B to C?*

---

### Step 6: Define Weights and Predicates (level ≥ 2)

Enrich relations only when a question in Step 5 needs it.

| Enrichment | Lives on | Shape | Example |
|------------|----------|-------|---------|
| **Weight** | edge | a number per dimension: time, cost, risk, delay, strength | a transition costs 30 and takes 15 |
| **Attribute** | node | a property the predicate reads | loudness, power draw, vibration sensitivity |
| **Predicate / threshold** | edge | `edge counts ⇔ structural edge exists ∧ predicate(attributes, limit)` | acoustic influence above 25 dB; thermal above 15 °C; electrical above 0.6 kW |
| **Semantics label** | edge | why the edge exists: finish-before-start, shared resource, data hand-over | explains a precedence; never changes the order |

**Rule:** a weight function is a *parameter of the question*, not of the graph. The same journey graph answers "cheapest" with the cost function and "fastest" with the time function, and the two can pick different routes. Never bake one dimension into the structure.

**Rule:** *"not every noise connects neighbouring laboratories, and not every shared power supply creates a conflict."* A predicate turns a broadcast influence into a real one. Document the limit and who sets it.

**What weights unlock:** simulation. Weaken an edge to see behaviour after degradation; strengthen it to see the effect of an investment (*"if we weaken the attenuation between A and B, we unlock 12 extra reservations per week"*). Note these as *unlocked, not required* unless the requirements ask.

---

### Step 7: Separate Semantic Layers (level ≥ 3)

When rules change faster than structure, or a relation "works on Monday, not on Wednesday", build **two graphs over the same nodes** and combine them. The trainers show two kinds of pair; a domain may have either, or both, and may have no policy layer at all.

| Layer | Meaning | Who owns it | Tempo |
|-------|---------|-------------|-------|
| **Topology graph** | what is physically or structurally possible: wants, routes, physical influences, precedences | the domain module | slow |
| **Policy graph** | what is allowed, desired or profitable: permissions, limits, closures, exceptions | the **client** of the graph module | fast |
| **Universal graph** | laws that hold everywhere: physics, regulation | reference data | slowest |
| **Local graph** | this building's, this route's quirks: shared supply, closed section, a driver's detour | the people who know the place | medium |

**Combination:**
- **Intersection (AND)** — an edge survives only if it is in both. Answers *"what works today"*: wants ∩ permissions = executable exchanges; routes ∩ restrictions = drivable today.
- **Union (OR)** — an edge exists if it is in either. Answers *"the full horizon"*: physics ∪ local infrastructure = real influence; official routes ∪ drivers' detours = everything we know.

**Rule:** *"Topology in one graph, policy in the other."* A new rule is a new condition when the client builds the policy graph. *"You do not touch the algorithm, you do not change the processing logic, you do not break the tests."* Never absorb a rule into the mechanism.

**Rule:** the policy graph knows nothing about limits, departments or tomorrow's exceptions. It only says allowed / not allowed per pair. The reasoning that produced each edge lives above it.

**Node-space alignment — a design decision.** Two graphs can be combined only over the same node identity. The trainers raise the case of a universal graph (process influences process) meeting a local one (process-in-room influences process-in-room) and leave it open. Options:
1. **Broadcast** — every universal edge applies between every pair of places (including a place with itself). Simple, conservative, over-connects.
2. **Adjacency** — a universal edge applies only between places declared adjacent (a place may be adjacent to itself). Needs an adjacency graph as a third input; adjacency is an edge like any other, so declare its direction: "A next to B" is symmetric only if both directions are declared.
3. **Explicit** — only declared local edges count; the universal graph is a checklist for authoring them.
Choose per domain and record it. Then union the local edges in.

**Rule (consequence):** a cycle or path found on an intersection is valid only if *every* hop survived. One missing permission kills the whole group, not one hop.


---

### Step 8: Ask the Second Questions (level ≥ 4)

Once a graph exists, the trainers' rule is to interrogate it further. Three pairings recur:

| Having found… | Immediately ask… | Structure | Value sign |
|---------------|------------------|-----------|------------|
| an **order** | *what can run in parallel?* | conflict graph (undirected, "cannot coexist") → minimal colouring; each colour is one environment / track / equipment set | fewer colours = fewer environments to provision; each colour class is a set of stages that may run together |
| a **zone** or **cycle** | *which node holds it together? what if it disappeared?* | articulation point | **domain-dependent** — see below |
| a **cycle that blocks** | *which edges create it?* | feedback arc set | removing = loss of possibility, or clean-up? |

**Rule (firm):** *"If you ask 'in what order', immediately ask 'what can run in parallel'."* Order is only half the truth: time constrains, but so do resources. The conflict graph is a separate graph from the precedence graph, over the same stages, with a different edge meaning.

**Rule (firm) — connector polarity flips with domain class:**
- **Infrastructure and flow** (power, water, logistics, money settlement): an articulation point is a *single point of failure*. Seek redundancy: protect the node or add alternative paths. More edges, less risk.
- **Information, business, social, coordination**: an articulation point is a *bottleneck*. Removing it, deferring it to the end of a batch, or moving it splits one big queue into independent batches that run in parallel. Fewer edges, more order. *"In the energy world it would be a blackout; in the laboratory world it is optimisation."*

**Discovery route:** this layer is found *bottom-up from the model*, not from language. Still listen for *"everything flows through X"*, *"it would be much faster if not for X"*.


---

### Step 9: Determine Boundaries and Deployment

**Who does what:**

```
Client / business rules layer:  builds the policy graph, evaluates transition conditions,
                                decides eligibility, supplies weights and limits,
                                chooses the weight function for the question
Graph module:                   builds topology from entity data, combines layers,
                                runs the mechanism, returns the structure
Consumer (use case):            applies the structure — executes the cycle, follows the path,
                                schedules the order, warns about the zone
```

The graph never decides whether something *should* happen. It reports what the arrangement *allows*.

**Deployment ladder** — pick the lowest rung that fits, in this order:

| Rung | When | Note |
|------|------|------|
| One file, hand-written | one or two mechanisms, local use | full control, no dependency |
| Library, in memory | several places or complex mechanisms | *"the mathematical level: in memory, no server, often no data migration"*; graph rebuilt from entities, or persisted small as plain documents |
| Graph database | data genuinely large **and** relations dynamic | rare; not implied by discovering a graph |
| Separate service | level 5 only: results shared across contexts, own life cycle, common rule-constrained global state | orthogonal to storage. **Name it in domain terms** ("deliveries", "settlement network"), never in graph terms |

**Rule:** *"Discovering a graph does not oblige you to install a graph database."* Default is a runtime structure derived from existing entities.

---

### Step 9.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:

- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred from a structural rule of the archetype (state the rule)
- **(X)** — neither: assumed silently

**Decision checklist:**

| Decision area | Example decisions to check |
|---------------|---------------------------|
| Complexity level | Is the chosen level justified by a signal? Are skipped layers marked? |
| Node identity | Which identity per graph? Re-projection when combining? |
| Edge meaning | One meaning per graph? Multiplicity? Time filter? |
| Direction | Kept or dropped, per graph, per question? |
| Cycle polarity | Desired or defect? Reject early or feedback arc set? First or all? |
| Execution of a structure | Release-then-assign? What about requests outside the cycle? |
| Unmatched events | No-op or error? |
| Weights | Which dimensions, on edges or nodes? Who supplies the weight function? |
| Predicates | Which limits, who sets them, which module evaluates? |
| Layers | Topology vs policy split? Intersection or union? Node-space alignment option? |
| Rule ownership | Does the client build the policy graph? Nothing rule-shaped inside the mechanism? |
| Second questions | Was "what in parallel" asked after "in what order"? Connector polarity stated? |
| Lifetime and deployment | Runtime or persisted? Rung of the ladder? Service named in domain terms? |

**For every (X) decision found:**

1. Low impact (technical, easily changed): mark as explicit assumption in Implementation Notes.
2. Affects business behaviour (cycle polarity, direction, node-space alignment, connector polarity, who owns policy): **stop and ask** using `AskUserQuestion` before delivering. Non-interactive: pick a default, mark (X), and list it first in Implementation Notes.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

```markdown
# Graphs Archetype Model: [Domain Name]

## Graph Domain
[One sentence: what is connected, what a connection means, what the business asks of the whole]
**Complexity level**: [0–5] — [signal that justifies it]

## Concept Mapping

| Domain Concept | Graph Archetype | Notes |
|----------------|-----------------|-------|
| ...            | ...             | ...   |

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
| Question | Answer | Source (R/A/I/X) |
|----------|--------|------------------|

## Graphs: Nodes & Edges

### [graph name]
| Aspect | Decision |
|--------|----------|
| Node | [identity] |
| Edge | [meaning; from → to] |
| Direction | directed / undirected — [why] |
| Multiplicity | ... |
| Lifetime | runtime from [entities] / persisted |
| Time filter | none / [rule] |
| Built by | [module] |

[Repeat per graph]

**Sketch** — a text or mermaid drawing of one representative instance: nodes, edges, direction. One per graph, or one combined drawing with the layers distinguished.

## Questions & Structures

| Business question | Graph | Structure | Mechanism | Precondition | On empty result |
|-------------------|-------|-----------|-----------|--------------|-----------------|

[Cycle polarity, first-vs-all, execution convention, unmatched-event handling stated here]

## Weights & Predicates
[omit if level < 2, or if nothing in the requirements enriches an edge or node — say which]
| Enrichment | Lives on | Dimension / limit | Supplied by | Used by question |
|------------|----------|-------------------|-------------|------------------|

## Semantic Layers
[omit if level < 3 — say so; if there is no policy layer, say so and show only the universal/local pair]
| Layer | Graph | Meaning | Owner | Combined with | Operation |
|-------|-------|---------|-------|---------------|-----------|
Node-space alignment: [same identity / broadcast / adjacency / explicit — why]

## Derived Questions
[omit if level < 4 — say so]
| Having found | Second question | Structure | Value sign (guard / remove) |
|--------------|-----------------|-----------|-----------------------------|

## Boundaries & Deployment
| Responsibility | Module |
|----------------|--------|
| builds policy graph | ... |
| evaluates conditions / supplies weights | ... |
| builds topology, combines, runs mechanism | ... |
| applies the structure | ... |
Deployment rung: [file / library / database / service] — [why]

## Worked Example
[A small concrete instance: nodes, edges, the question, the structure found, the effect applied — checked by hand]

## Implementation Notes
[Key decisions, assumptions for unanswered questions, layers deliberately not modeled, edge cases]
```

---

## Common Patterns & Pitfalls

### Pattern: Change the Data, Not the Algorithm

Every business rule about *who may connect with whom* is an edge in a policy graph built by the client. The mechanism (cycle detection, ordering, components) is never edited to absorb a rule. When a new rule arrives, add a condition where the policy graph is built and re-run the same mechanism on the intersection. If you find yourself adding a parameter to the algorithm, a rule has leaked.

### Pattern: One Meaning per Edge, One Graph per Meaning

Precedence ("before") and conflict ("cannot coexist") are two graphs over the same stages, not two edge types in one graph. Wants and permissions are two graphs over the same nodes. *"In the world of graphs you add a second perspective, not an exception."* Typed edges with flags and exceptions are the entity-model habit reappearing.

### Pattern: The Question Chooses the Direction

Keep direction in the base graph when it carries meaning, and let each question decide: "whom must I inform" reads arrows; "who has to synchronise with whom" ignores them; "in what order" requires them. Do not build separate directed and undirected copies by hand.

### Pattern: Structure First, Then Add or Remove

Detect a structure, then perturb it: remove the connector to split a batch, remove the feedback edges to make a process orderable, weaken an edge to simulate degradation, add an edge to see what the union unlocks. This is where *"the graph stops being theory and becomes a tool for generating value"*.

### Pitfall: Brute Force with Snapshots

A loop that copies the current state, simulates a move, tracks who became "deficit", scans for a matching request, and restarts from the next candidate is a cycle search written by hand. It is correct for small inputs and unmaintainable at the first rule change. Name the cycle instead.

### Pitfall: Flag Soup Instead of States

`hasLatePayment`, `promotionActive`, `campaignActive`, `interestBelowThreshold`... *"There is no longer a customer state. There are only dozens of variables which together form an arrangement nobody understands."* The tell: the auditor asks what must be true for the 10% discount and nobody can answer in one sentence. Replace with states, transition conditions and path queries.

### Pitfall: Forcing the Archetype on a Single Case

*"It is an archetype when the pattern repeats naturally in different places of the business and easily generates additional value. It is overengineering when you force it where there is only a single case."* Level 0 is a valid, complete answer.


---

## Quality Checks

Before returning the model, verify:

- [ ] Complexity level is stated with the signal that justifies it; skipped layers are marked *deliberately not modeled*
- [ ] Every graph has a named node identity, one edge meaning, an explicit direction decision and a lifetime
- [ ] At least one graph is drawn, not only tabulated
- [ ] Graphs that are combined share a node identity, or the alignment option (broadcast / adjacency / explicit) is stated
- [ ] Every business question maps to exactly one structure and one mechanism, with its precondition and empty-result behaviour
- [ ] Cycle polarity is explicit wherever cycles can appear; acyclicity is stated as a precondition of ordering
- [ ] "In what order" is accompanied by "what in parallel", or its absence is justified
- [ ] Connector polarity (guard vs remove) is stated for any articulation-point question
- [ ] No business rule lives inside a mechanism; the policy graph is built by the client
- [ ] Weight functions are parameters of questions, not properties of the graph
- [ ] Concept mapping table is present and complete; Unmapped Concepts section is present (even if empty)
- [ ] All clarifying question answers (or assumptions) are reflected in the model
- [ ] Worked example is hand-checked: the structure found matches the edges listed, and the applied effect is consistent

---

## Example

**Input:** "Users book slots (calendar appointments). One slot has at most one owner. A user can ask to move to another slot. If the target is free, move immediately. If it is taken, the request waits. Periodically the system looks for groups of users who can swap so that everyone in the group gets what they asked for. Users from different companies may not swap with each other, and there is a limit on swaps between specific pairs of users."

**Detected level:** 3 — a cycle question (level 1) with permission rules that change independently of the wants (level 3). No weights are mentioned (level 2 skipped). Parallelism and connectors not asked (level 4 skipped).

(Rendered as a non-interactive run: `AskUserQuestion` was unavailable, so the one material (X) decision is defaulted, marked, and listed first in Implementation Notes, per Step 9.5.)

**Output:**

```markdown
# Graphs Archetype Model: Slot Swap Waiting Room

## Graph Domain
Users hold slots and ask to move to other slots; the business asks which pending requests can execute together, subject to swap permissions.
**Complexity level**: 3 — individually impossible requests can succeed as a group (cycle), and eligibility rules (company, pair limits) change faster than the wants.

## Concept Mapping

| Domain Concept | Graph Archetype | Notes |
|----------------|-----------------|-------|
| Slot | Node (wants graph) | the contested resource; one owner at a time |
| User / owner | Node (permissions graph) | identity the rules speak about |
| Pending change request | Edge (wants graph): current slot → desired slot | a "want" |
| Request for a free slot | filtered out before the graph | executes immediately |
| "Can swap together" | Structure: cycle | the executable group |
| Company rule, pair limit | Policy graph: allowed transfers owner → owner | built by the client |
| Executing a group | Consumer applies the cycle | release all, then assign all |
| Waiting room | the set of edges of the wants graph | rebuilt per run |

## Unmapped Concepts
- Slot capacity or price — none mentioned; would belong to accounting or pricing, not here.

## Clarifying Questions & Answers
| Question | Answer | Source (R/A/I/X) |
|----------|--------|------------------|
| Is a cycle desired or a defect? | Desired — it is the swap group | R |
| First cycle or all cycles? | First is enough; no priority between groups mentioned | X (low impact) |
| Who builds the permissions graph? | The client (company + limit rules) | I — rule ownership |
| Direction of permission? | Directed: A may give to B does not imply B may give to A | X (material, assumed; confirm) |
| Requests outside the found cycle? | Left untouched, retried next run | I — cycle execution |

## Graphs: Nodes & Edges

### wants
| Aspect | Decision |
|--------|----------|
| Node | slot |
| Edge | change request, from the requester's current slot → desired slot |
| Direction | directed — the request has a giver and a taker |
| Multiplicity | one request per user; several users may want the same slot |
| Lifetime | runtime, rebuilt from pending requests and current slot owners each run |
| Time filter | none |
| Built by | swap module |

### wants (owner projection)
| Aspect | Decision |
|--------|----------|
| Node | owner |
| Edge | the same request, re-projected: current owner of the from-slot → current owner of the to-slot |
| Direction | directed |
| Multiplicity | one edge per (from-owner, to-owner) pair per run |
| Lifetime | runtime, derived from the wants graph plus current ownership |
| Time filter | none |
| Built by | swap module, from the wants graph and current ownership |

### permissions
| Aspect | Decision |
|--------|----------|
| Node | owner |
| Edge | "owner A may transfer to owner B" |
| Direction | directed, revocable at any time |
| Multiplicity | one edge per ordered owner pair |
| Lifetime | runtime, built by the client from company membership and pair-limit counters |
| Time filter | none — permissions are evaluated as of the run |
| Built by | client (business rules layer) |

Sketch (owner projection ∩ permissions, the 5-slot instance below):

```
Alice → Bob → Charlie → Diana → Eve
  ↑                               │
  └───────────────────────────────┘
```

## Questions & Structures

| Business question | Graph | Structure | Mechanism | Precondition | On empty result |
|-------------------|-------|-----------|-----------|--------------|-----------------|
| Which pending requests can execute together, respecting the rules? | wants (owner projection) ∩ permissions | cycle | cycle detection on the intersection | one slot, one owner | no group this run; nothing changes |

Cycle polarity: desired. First cycle only. Execution: release every from-slot in the cycle, then assign every to-slot. Requests not on the cycle are untouched.

## Weights & Predicates
Omitted — level < 2. No priority, cost or age of requests was mentioned.

## Semantic Layers
| Layer | Graph | Meaning | Owner | Combined with | Operation |
|-------|-------|---------|-------|---------------|-----------|
| topology | wants (owner projection) | who wants whose slot | swap module | permissions | intersection (AND) |
| policy | permissions | who may give to whom today | client | — | — |
Node-space alignment: same identity (owner) after re-projecting wants from slots to their current owners; the slot graph alone cannot be intersected with owner-level rules.

## Derived Questions
Omitted — level < 4. Candidate, not requested: "which single request connects two otherwise independent swap groups?" (connector; here a bottleneck to defer, not a failure to guard).

## Boundaries & Deployment
| Responsibility | Module |
|----------------|--------|
| builds permissions graph | business rules layer (company, pair limits) |
| builds wants and its owner projection, intersects, finds the cycle | swap module |
| applies the cycle (release, assign), persists slots | reservation use case |
Deployment rung: one file, hand-written — a single mechanism plus an intersection, used in one place, rebuilt per run from slots and requests. Move to a library only if a second mechanism (ranking, parallelism) is added or a second context needs the same network.

## Worked Example
Slots A–E owned by Alice, Bob, Charlie, Diana, Eve. Requests: Alice A→B, Bob B→C, Charlie C→D, Diana D→E, Eve E→A.
Owner projection: Alice→Bob, Bob→Charlie, Charlie→Diana, Diana→Eve, Eve→Alice.
Permissions: exactly those five directed pairs.
Intersection keeps all five edges; cycle detection finds the 5-edge cycle; all 5 requests execute.
After release-then-assign: A=Eve, B=Alice, C=Bob, D=Charlie, E=Diana.

Counter-case: slots A, B, C owned by Alice, Bob, Charlie; requests Alice A→B, Bob B→C, Charlie C→A; permissions Alice→Bob and Bob→Charlie only. Intersection drops Charlie→Alice; no cycle; 0 requests execute; ownership unchanged.

## Implementation Notes
- Permission direction assumed asymmetric (X, material) — confirm; if symmetric, add both edges when building the policy graph, not a flag in the mechanism.
- First-cycle choice means fairness between competing groups is not modeled (X, low impact); revisit if priorities appear (then level 2: weight requests by age).
- The pair-limit counter lives above the graph; it is read when building permissions, and updated by the reservation use case after execution.
- Requests targeting a free slot never enter the graph.
- Deliberately not modeled: weights (level 2), derived questions (level 4), a shared network capability (level 5).
```

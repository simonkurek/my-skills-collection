---
name: party-archetype-mapper
description: Transform domain requirements into a Party Archetype model. Identifies party kinds and one identity space, registered identifiers with assignment policies, roles (classification, relationship roles, role catalog), the address book, named directed relationships, scoped capabilities with role requirements, and aggregate/integration boundaries. Adopts only the layers the domain signals.
argument-hint: "[domain requirements or feature description]"
---

# Party Archetype Mapper

Transform any domain description that involves people and organizations — who they are, what they count as, whom they are linked to, and what they can do — into a Party Archetype model. The archetype separates **identity** (stable) from **roles** (unstable), and pushes everything that merely orbits identity (contact points, relationships, capabilities) into its own consistency boundary.

**Output goal**: An implementable model that gives the system one truth about *who* it serves, roles that change without redeployment, relationships that carry meaning and direction, and capabilities the system can reason about — adopted only to the depth the domain needs.

## When to Use

- The same real-world party plays several roles, is sometimes a person and sometimes an organization, or exists without being a system user
- Parties are linked by named relationships (employs, commissions, represents, supplies, belongs to), or the business asks *who can do X, where, when, how much*
- A "User" table is silently becoming the organization's party model

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the Party archetype does not fit, and briefly explain why.

### The core question

> *"Do different parties (people and organizations) appear in this system, play more than one role, enter into relationships with each other, or change role depending on context?"*

If **yes** to any → Party likely fits.
If every party is exactly one natural person with a login and nothing else → a plain user model is adequate. Do not map.

**Counter-questions that route elsewhere:**
- *"How much X does S have?"* → accounting archetype (Party may still supply *who* S is)
- *"What does it cost?"* → pricing archetype
- *"Who may log in, with which permissions on which screen?"* → identity & access management. Party answers "does X hold role Y"; it does not hold credentials
- *"Is the slot free on Wednesday at 10:00?"* → booking / availability. Party's capability says what is *possible*, never what is *free*
- *"What state is the ticket in?"* → state machine

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "customer is sometimes a person, sometimes a company, sometimes both" | ✅ Yes |
| "the same company is our supplier and our customer" / "employee who is also a customer" | ✅ Yes |
| partner, supplier, subcontractor, branch, facility, team; employs, commissions, represents, cooperates | ✅ Yes — bridge words |
| "customer without an account" (import, B2B API, lead) | ✅ Yes — a user without a user |
| tax number, national id, passport with expiry, licence number | ✅ Yes — registered identifiers |
| "who can perform X at location L on day D, max N per day" | ✅ Yes — capabilities |
| "every user is one person with a login; no companies, no non-user customers" | ❌ No — plain user model |
| "password, permissions per screen, sessions" | ❌ No — identity & access; Party supplies roles only |
| "how many points / how much credit does the customer have" | ❌ No — accounting |
| "is the doctor free at 10:00" | ❌ No — availability, not capability |
| "org chart" | ⚠️ Borderline — parent pointers only, forever? A tree suffices. Siblings, matrices, a person in two organizations? Fits |
| "contact book" | ⚠️ Borderline — do the owners play roles or relate to each other? If not, keep a contact model |

**Two more borderline tells:** when the answer to "can a user be a company?" is *"theoretically yes, but it shouldn't"*, the model is already failing — fits. A sole trader is not a new kind; it is a person who also carries organizational identifiers — fits at the official-identity layer.

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The Party archetype requires parties that play several roles, relate to each other, or change
role by context. This domain is a [single-kind user model / access-control model / booking model / ...] because:
- [specific reason from the requirements]
- The natural question is "[who may log in / is the slot free / ...]", not "who is who, in what role, linked how?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — which people and organizations appear, what they count as, how they are linked, and what the system must be able to say about them?"

---

### Step 1: Assess Complexity Level

Identify **every layer that has a signal** in the requirements. Level 1 is the foundation; levels 2–7 are independent of each other above it — a domain may need relationships without an address book, or capabilities without registered identifiers. Model each layer with a signal, **and no further**. Report the level as the highest one modeled, but adopt layers by signal, not by number.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 0 | **Single-kind users** | Every party is one natural person with an account; no companies, no non-user customers | (none — a plain user model is adequate) |
| 1 | **Identity & classification** | People *and* organizations; some parties hold several roles; some have no account | flags (`isCustomer`, `isPartner`), nulls as a modeling device, one class per role |
| 2 | **Official identity** | Tax numbers, national ids, passports, licences — with their own validation, applicability rules, some expiring | a semantic number used as primary key |
| 3 | **Contact points** | Many addresses of many kinds (postal, e-mail, phone, web) with uses and validity; rules over the set ("one active billing address") | address columns on the party; `BillingAddress` / `ShippingAddress` classes |
| 4 | **Network** | Named, directed links between parties with rules: employs, supplies, belongs to, represents | `parentId`, a flat link table with no meaning |
| 5 | **Governed roles** | Roles are added, changed, retired without deployment; a role carries constraints; typos have legal consequences | free-text role names drifting; a class per role |
| 6 | **Operational capability** | "Who *can* do X, where, when, how many, at what level, under which certificate"; role eligibility depends on it | tags, text fields, hundreds of `if`s |
| 7 | **Organizational graph** | Multi-hop questions: partner's partner, cycles, who is a single point of failure, silos | JOINs across half the screen |

**Guidance — which steps to run:**
- Level 0: the archetype is overkill. Emit the ❌ Does Not Fit block from "When NOT to Use" and stop; do not produce a model.
- Always: Steps 0–3, Step 4 (party kinds & identity), Step 6 (roles), Steps 10–11.5.
- Add one step per signalled layer: level 2 → Step 5; level 3 → Step 7; level 4 → Step 8; level 5 → the role-catalog part of Step 6; level 6 → Step 9; level 7 → the graph part of Step 10. Graph questions traverse relationships across parties, so level 7 implies autonomous relationships in Step 8.

Partial adoption is normal — the archetype is *"a way of thinking; pick the elements that fit your business."* Mark skipped layers in Implementation Notes as *deliberately not modeled*.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps between the requirements and the archetype. Ask about **two categories** of questions in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Ask only about layers Step 1 selected, and only the questions whose answer changes the **shape** of the model — typically role representation, the placement decisions, and consistency mode. Take the simplest option for the rest and mark it *(assumed)*.

#### Category A — Standard Party decisions

Ask only about those **not clearly addressed** in the requirements. Frame them as design choices:

- **Party kinds**: People only, organizations only, or both? Do internal units (department, branch, team) need to be told apart from independent companies?
- **Role representation**: Plain role names / a fixed set defined in code / a governed catalog with constraints? (See Step 6 for the fit rules.)
- **Roles needed**: Classification only ("this party is a supplier") / roles inside relationships only / both?
- **Contact points**: Inside the party / in a separate address book per party?
- **Relationships**: Inside the party / autonomous relationship objects / not needed (classification is enough)?
- **Consistency of cross-party rules** ("a manager has at most 10 subordinates"): Immediate — lock both parties / eventual — accept and reconcile / it depends?
- **Capability placement**: Inside the party like roles / own aggregate like addresses / it depends?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the archetype supports but the requirements do not mention**. Ask only where the requirements hint that the dimension exists. Do not limit yourself to this list — reason freely:

- **Identifiers**: Do any expire or get renewed (passport, licence)? Must a value be unique across all parties? Is applicability keyed on party kind, on role ("customer number only with the customer role"), or on other identifiers already held?
- **Role validity**: Do role assignments have a start and an end?
- **Role eligibility**: Should assigning a role be *gated* by requirements (age, capability, certificate), or is eligibility an advisory query?
- **Symmetric relationships** (partnership, friendship): One relationship with the same role on both sides, or two directed ones?
- **Relationship rules**: Quantitative (max N), qualitative (only organizations employ), temporal (not terminable before 3 months)?
- **Address-book rules**: Exactly one active address per use? Uses restricted by party kind? Change-frequency limits? Maximum count?
- **Point-in-time data**: Do consumers (orders, contracts, invoices) need party data *as of a moment*, unaffected by later changes?
- **Party lifecycle**: Is a blocked / merged / archived party needed? (The archetype sources do not model it — if wanted, it lands in Unmapped Concepts or a lifecycle layer above Party.)

Collect answers before proceeding. If the user cannot answer, document the assumption made in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

Always include **"To zależy / It depends"** as an explicit option in every `AskUserQuestion` call — do not rely on the automatic "Other" fallback. Place it as the last option in each question. If the user selects it, treat it as a **variable policy**:

- Name the *policy* the model will accept (identifier-defining policy, role-defining policy, relationship-defining policy, address-defining policy)
- Note in **Implementation Notes** that its concrete rules are supplied from outside (a policy factory, configuration, tenant settings) and the model only *enforces* what it receives
- Do **not** model the decision logic inside the Party model

This is the correct outcome — *"data is dumb but stable; rules are smart but changeable."* Variability lives in policies, not in the party.

#### If `AskUserQuestion` is not available

Do not stop. Use these defaults, list each in **Clarifying Questions & Answers** marked *(assumed)*, and treat every Category B gap as *not wanted* unless the requirements imply it: plain role names; address book, relationships and capabilities as separate aggregates (the sources' choice) for every layer that is modeled; eventual consistency with reconciliation; no role validity; duplicates are no-ops.

---

### Step 3: Map Domain Concepts to Party Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Party Archetype | Notes |
|----------------------|-----------------|-------|
| [domain noun/verb]   | Party (Person / Organization / Unit) / Registered Identifier / Role / Role Type / Relationship / Address / Capability / Operating Scope / Defining Policy | [why] |
```

An adjective-qualified noun ("corporate customer", "premium member") is usually a role or a relationship on an existing kind, not a new kind.

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear Party archetype equivalent:
- [concept] — [reason it doesn't fit / which module owns it]
```

This section must be present even if empty (`None identified`). Typical residents: credentials and sessions, availability and booking, contracts and their duties, medical or financial records, party merge/golden-record processes.

---

### Step 4: Define Party Kinds and Identity

Decide **which kinds of party exist** and what is shared by all of them.

| Concept | Meaning | Key rules |
|---------|---------|-----------|
| Party | The identity core: party identifier + registered identifiers + roles + version | *"Not an accidental aggregation of data — the smallest coherent unit"*; roles and identifiers are authorization-critical, so their changes are atomic |
| Person | A natural person: customer, employee, patient, system user | Carries only person data (names) |
| Organization | A legal or organizational entity | Carries only organization data (name) |
| Organization unit | An internal part: department, branch, team, facility | Belongs to a company **through a relationship**, never a parent field |
| Company | An independent economic entity | Specialise further (legal forms) **only where rules differ** |
| Party identifier | One technical id space for all kinds | No per-kind ids; never a semantic number |

**Rule (one identity):** *"There is no per-kind id. There is one id for all party kinds."* Roles, identifiers and relationships are kind-agnostic; one id space is what lets them be.

**Rule (no semantic keys):** A tax number, national id, phone or e-mail is never the key. They are unique only inside one slice of reality, get reassigned, are shared, and stop being unique in the next market.

**Rule (own data only):** each kind holds only the data that makes sense for it. A person has no company name; a company has no first name. Null is *"not missing data but a missing decision."*

**Convention:** generate the id in the application (unit-testable, extensible with prefixes or namespaces); a random UUID is usually enough. Alternative: database-generated.

**Records into:** the Party Kinds & Identity section of the Output Format.

---

### Step 5: Define Registered Identifiers and Their Policies

*Omit if the domain shows no official-identity signal — say so.*

A **registered identifier** is how the outside world names a party: *"nobody asks for your party id; they ask for your tax number."*

**Each identifier type has:**
- **type** — what it is (tax number, national id, passport, licence, customer number)
- **value** — its textual form, validated by the type's own rules (format, checksum)
- **validity** — some never expire (national id, tax number); some do (passport, licence). *"Not all identifiers are eternal; without time-awareness the model becomes a paper model."*

**Applicability is a policy, not a field.** For each identifier type decide **which parties may hold it**, keyed on party kind ("a company cannot have a national id"; "a person *can* have a tax number" — the sole trader), on role ("a customer number for anyone holding the customer role"), or on identifiers already held ("if you have X you cannot have Y").

**Rule (enforced in the model):** the applicability check is a constant part of attaching an identifier, so the party enforces it — never only an application service, where it reads as use-case-specific and gets forgotten.

**This is a design decision — where the rule text lives:**
1. inside the identifier type ("can I be assigned to this party?") — fine for small systems keyed only on kind; cannot vary per tenant, and cross-identifier rules create cycles
2. one type-based policy — works, changes mean touching it
3. **many small policies composed** — preferred by the sources when rules vary; supplied by a factory when the choice depends on tenant or region, hard-wired otherwise

**Convention (uniqueness):** an identifier value is unique within one party by construction; uniqueness *across* parties is a domain decision (Step 2) enforced at write time if wanted.

**Records into:** the Registered Identifiers & Policies section of the Output Format.

---

### Step 6: Define Roles

Roles are the archetype's centre-of-gravity inversion: **roles are data, not types.** *"Roles do not define parties; parties possess roles."* A role is a trait assigned at a moment: added, removed, changed, with an event for each.

**Two forms of role — decide which you need:**

| Form | Meaning | When it suffices |
|------|---------|------------------|
| Global (classification) role | What a party *is* on its own: customer, supplier, VIP | When the only relationships that matter are from your organization outward; classification is *"simpler, faster and precise enough for most cases"* |
| Role in a relationship | What a party is *toward another party*: employee/employer, patient/care provider | When relationships between third parties matter (Step 8) |

The same name may appear in both forms — a global `patient` role that classifies, and a `patient` side of a care relationship. When it does, say which answers "is X a patient?" (normally the global role) and make the relationship side *require* it rather than duplicate it. Two sources of one fact will drift.

**Rules:**
- A party holds zero or more roles at once; each role instance belongs to one party
- Roles accumulate and are removed explicitly — a party promoted into a new role keeps the old one until something removes it. Nothing auto-retires a superseded role unless the domain says so
- **Role applicability is a policy**: "B2B customer" not for persons; "employee" not for a department; only organizations employ; only persons are employees, patients or consumers. Check it at assignment, and inside the factory that builds a relationship side
- Role rules change faster than identifier rules — a policy factory, or a role factory that embeds the policies, is a good fit

**This is a design decision — how roles are represented** (absent a signal, plain names):

| Strategy | Shape | Fits when | Breaks when |
|----------|-------|-----------|-------------|
| Plain name (value object) | An immutable name; add = put in the set after the policy check | MVP, simple systems, instant delivery | Names drift ("Client", "client", "klient"); no metadata; reporting pain — *"live fast, die young"* |
| Fixed set in code (type hierarchy) | One type per role, each knowing which parties it serves; the policy is the type | 2–3 stable roles; errors have legal or financial consequences; self-describing code required | Roles appear and disappear dynamically — a new role is a release; *"agility disappears, concrete remains"* |
| Governed catalog (role type + role instance) | A catalog of role types (name, description, constraints, requirements) and instances that reference them | Large, dynamic, multi-tenant; roles change without deployment | Centralization: changing a type changes every instance |

*Level 5 unlocks the catalog.* A role type is *"the label on the yoghurt"*: name, description, constraints (e.g. doctor ≥ 35 years old), and — at level 6 — requirements against capabilities (Step 9). Anything shared by all holders of a role attaches to the type once, not to hundreds of instances.

**Convention (role validity):** the sources treat a role as *"a trait at a given moment"* and the fuller role shape carries a validity, but the reference implementation does not. Decide in Step 2; default to no validity unless the domain asks "who *was* a manager in March".

**Records into:** the Roles section of the Output Format.

---

### Step 7: Define the Address Book

*Omit if the domain shows no contact-point signal — say so. A distance or location in a requirement ("within 50 km") is usually a capability scope (Step 9), not an address.*

An **address** is *"any point of contact or spatial identification — somewhere you can send to, call, write to, or reach"*: postal, e-mail, phone, web, geo-coordinates. All kinds are equal-rank: same lifecycle, same contract. *"An address is not a field. It is a separate entity with identity, logic and rules."*

**Each address has:**
- its own identity and the owning party's id
- **kind** — what it is (postal, e-mail, phone, web); each kind has its own data and validation
- **uses** — what it is *for* (billing, shipping, mailing, residential, contact, legal, support…); one address may carry several uses
- **validity** — its period; succession over time is normal (billing address 2020–2023, then 2023→)
- lifecycle events (defined, updated, removed; extensible), emitted per kind because each carries a different delta

**Rule (kind × use are orthogonal):** the kind says *what it is*, the use says *what it is for*. Never create `BillingAddress` / `ShippingAddress` classes — that is the combinatorial explosion again. New uses are added without touching existing logic.

**Rule (the set is the boundary):** the rules people actually want are rules over the **collection** — "exactly one active contact address", "billing only for organizations", "shipping address changed at most once a quarter". So the consistency boundary is the **address book**: all addresses of one party, keyed by the party id, enforcing its policies itself. *"All policies land here. Not the party, not the address — the address book, because it knows the whole context."*

**This is a design decision — where the address book lives:** inside the party (intuitive, *"often enough in small systems"*; cost: one wide transaction where verifying an address blocks changing a marketing consent) or a separate aggregate keyed by party id (chosen by the sources; the party knows nothing about addresses; extractable to a module or service; cost: two transactional contexts and event synchronization — *"where addresses are trivial, a cannon at a fly"*).

**Address-defining policy** answers one question: *may this address be defined, added or updated?* Rule categories: no duplicates of the same kind; no overlapping validity for the same kind and use; use restricted by party kind or role; quantity limits; contextual rules. Policies compose.

**The same boundary test applies to everything orbiting the party** — preferences, consents, security settings. Ask the five questions: *Does it co-constitute identity? Must its change be immediately consistent with the rest? How often does it change, and independently? What rules govern the change? Can operations on it run in parallel?* *"Identity is the heart of the model — but heart does not mean pack ox."*

**Records into:** the Address Book section of the Output Format; trace each rule to its source using the codes of Step 11.5.

---

### Step 8: Define Relationships

*Omit if the domain shows no network signal — say so.*

A relationship is **not a link between two records; it is a domain object with semantics.** The four levels of expectation, in order:

1. **Semantics** — a **name** that carries meaning: employment, care, contract, membership. *"Without semantics the system does not know how to behave."*
2. **Direction** — asymmetric in practice. *"Every symmetric relationship can be built from two asymmetric ones; the reverse does not work."* Not always hierarchical: siblings need no father.
3. **Roles of the sides** — each side is a **(party, role)** pair: who is active, who initiates, who bears the consequences. Role applicability policies apply to each side.
4. **Rules** — quantitative (at most 10 subordinates), qualitative (only a company employs; doctor–patient only if both hold the roles), temporal (not terminable before 3 months)

**Each relationship has:** its own identity (so it can be tracked, archived and analysed independently of both parties), a name, a from-side and a to-side (each party id + role), and a validity (from, to — open-ended allowed: "active since today, no end").

**Rule (hierarchy is relationships):** a department belonging to a company, a team belonging to a department, a branch of a bank — all are ordinary named relationships ("organizational membership"), not a parent pointer. A three-level hierarchy is two relationships.

**Rule (where rules live):** the temporal rule is carried by the relationship record itself; qualitative and quantitative rules by a **relationship-defining policy** consulted when the relationship is created.

**This is a design decision — where relationships live:** inside the party (one query for all of a party's links; immediate consistency; *"good to start for simple systems with few relationships"*; breaks when the party becomes a god object and qualitative rules lock two parties in one transaction) or an autonomous aggregate (chosen by the sources; each relationship references both parties by id; scales and stays available; cost: cross-party rules can be violated between transactions).

**This is a design decision — consistency of cross-party rules:** *"What happens if a manager ends up with 11 subordinates instead of 10?"* The sources answer: rare and repairable — reconcile asynchronously (*"write a job that cleans it up"*), because *"scalability, flexibility and availability are in practice far more important than strict immediate consistency."* Choose immediate consistency only where a violation is unacceptable even briefly.

**Convention (symmetric relationships):** two directed relationships (source guidance) or one relationship with the same role on both sides (reference implementation). Choose one and state it.

**Convention (validity enforcement):** the reference implementation stores relationship validity without enforcing it. Decide whether validity is enforced at query time ("current relationships"), at write time, or by reconciliation.

**Records into:** the Relationships section of the Output Format.

---

### Step 9: Define Capabilities and Role Requirements

*Omit if the domain shows no capability signal — say so.*

Instead of modeling what someone *must* do (responsibilities — surveyed by the sources and declined: *"they stay in the books, and an empty role structure stays in the systems"*), model what someone **can** do. *"A responsibility is an obligation; a capability is a possibility."*

**Capability** — a discrete thing a party can do, held by the party, with:
- a **capability type** — a definition of meaning, not a label. *"For one person 'programming' is Python in data science, for another Java on the backend, for a third PLC drivers — one string becomes a bottomless bag."* The type declares **which operating scopes are mandatory**; a capability missing a mandatory scope cannot be created — the model is self-verifying
- **operating scopes** — *"a capability without context is an empty promise."* Small immutable value objects, each answering one question. Use only the scopes the type declares:

| Scope | Question | Example content | Satisfaction rule (held vs required) |
|-------|----------|-----------------|--------------------------------------|
| Location | where | a named facility; Warsaw + 50 km | a required *point* is covered if it lies inside the held area; a required *area* only if the held area contains it — say which question you are answering |
| Temporal | when | Mon–Fri 08:00–16:00 | held days and window cover the required ones |
| Quantity | how many | max 15 patients per day | held limit ≥ required, same period; "unlimited" always passes |
| Skill level | how well | an ordered scale the domain defines (named grades, 9/10, a certificate) | held level ≥ required |
| Protocol | by what rules | procedures, standards, certifications (prenatal ultrasound, ADR, Scrum) | held set ⊇ required set |
| Product | with what | product categories, max 500 kg | held set ⊇ required set |
| Resource | on what | a 3.5 t van; a team of 8 | held set ⊇ required set |

- a **validity** — an expired capability simply does not count; no separate "expired" state is needed

**Rule (possibility, not availability):** *"This is not a booking system."* Whether Wednesday 10:00 is *free* is another module's job. Capability answers *can*, never *is available*.

**This is a design decision — capability placement:** inside the party (like roles — natural when role rules depend tightly on capabilities) or its own aggregate (like the address book — natural when a party's capabilities have independent lifecycles such as licence renewals). The sources leave it open; the deciding factor is whether any rule needs immediate consistency across all of one party's capabilities.

**Role requirements — joining roles to capabilities.** A role type (Step 6) declares what a party must be able to do to hold it: *"Backend Delivery Team does not say 'a team of backend people'. It says: a team holding an active Backend Development capability at level ≥ 8, with AWS and Kubernetes certificates, working by Scrum."*

- **Role requirements** = a list of **capability requirements**; every one must be satisfied (AND)
- **Capability requirement** = a required capability type plus zero or more **scope requirements**; satisfied by *any* currently valid held capability of that type whose scopes satisfy every scope requirement (ANY across held, AND across scope requirements). No scope requirements → any valid capability of the type suffices
- A role type with no requirements is open to anyone; write `Role type: none` when no role is capability-gated
- The diagnostic **"what is missing"** — the unsatisfied capability requirements — is as valuable as the yes/no

**This is a design decision — enforcement:** the sources say the role *"is simply not granted because the formula is not satisfied"*; the reference implementation offers eligibility as a query without wiring it into assignment. Choose: gate at assignment, advisory query, or both (gate now, re-check by reconciliation when capabilities expire).

**Records into:** the Capabilities & Role Requirements section of the Output Format.

---

### Step 10: Define Boundaries and Integration

**Aggregates.** Restate the consistency boundaries chosen in Steps 4–9, each referencing the others **only by party id**:

| Aggregate | Contents | Own version / events |
|-----------|----------|----------------------|
| Party | id, registered identifiers, roles, kind-specific data | yes |
| Address book (per party) | all addresses of one party | yes, if separate |
| Relationship (each) | name, two sides, validity | yes, if autonomous |
| Capability (each or per party) | type, scopes, validity | yes, if separate |

**Convention (idempotent registration):** re-adding a role, identifier or address that is already there, or removing one that is absent, is a **no-op recorded as such**, not an error — re-submitting the same form twice is not a failure. Genuine failures are policy rejections and missing parties. Alternative: reject duplicates as errors.

**Placement in the architecture.** Party is generic across industries, so it sits **upstream** (strategic DDD), exposed as an **open-host service**: a module with its own store, contract and owning team — *"not a library, an architectural product."* A shared library degenerates into a shared kernel: every model change is a release wave, and a library cannot hold history, audit, retention or the right to be forgotten.

**Consumers.** *"Every business process starts with the question: who? Party answers exactly that"* — and nothing more. Four consumer shapes recur: processes that ask *who is who* (CRM: who is a customer, who represents whom — the CRM **delegates the write** of party data to Party and keeps its own processes); processes that need party data **as of a moment** (orders, contracts, invoices keep a **snapshot**; a guest order never creates a party; a later address change must not alter a placed order); authorization ("does X hold role Y"; approval chains derived from relationships); and operations that ask *who can do what, where, when* (capabilities — availability stays with them).

**Graph view** (*level 7 only*): every party is a vertex, every relationship an edge, every capability a vertex attribute. Name the questions the domain needs — cycles and back-links (A commissioned B which cooperates with A), indirect dependencies (which teams lose access if this branch closes), articulation points (whose departure disconnects key customers), connected components (silos). Divergent non-functional needs across consumers are a scaling decision above the model (separate instances, a fork, a graph database) — record the need, do not model it here.

**Records into:** the Boundaries & Integration section of the Output Format.

---

### Step 11: Write the Worked Queries

Pick 3–8 questions the business actually asks — the ones that motivated the mapping. Answer each **from the tables only**, and name the concept that decides it (a scope value, a role, a relationship, a validity date). Include at least one *no* answer and one question the model must refuse as out of scope (availability, credentials). If a query cannot be answered from the tables, a layer is missing or under-specified — go back. If a required answer conflicts with a scope's satisfaction rule, document the reinterpretation as an (X — confirm) decision rather than bending the rule silently.

---

### Step 11.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:
- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred from a rule in this skill (cite the step)
- **(X)** — neither: assumed silently

Walk every Category A and Category B question from Step 2, every block marked **This is a design decision** or **Convention** in Steps 4–10, and every rule in the address-book, relationship and role tables.

**For every (X) decision found:**

1. If the decision has low impact (purely technical, easily changed): mark as explicit assumption in Implementation Notes.
2. If the decision affects business behavior (which parties may hold a role, consistency mode, eligibility gating, snapshot semantics): **stop and ask** using `AskUserQuestion` before delivering the model. Without `AskUserQuestion`, deliver with the assumption marked **(X — confirm)**.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

```markdown
# Party Archetype Model: [Domain Name]

## Complexity Level
Level [n] — [name]. Layers modeled: [list]. Deliberately not modeled: [list].

## Concept Mapping

| Domain Concept | Party Archetype | Notes |
|----------------|-----------------|-------|
| ...            | ...             | ...   |

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
- [question] → [answer or "(assumed) …"]

## Party Kinds & Identity

| Kind | Own data | Examples |
|------|----------|----------|

Identifier: [one id space; generation choice]

## Registered Identifiers & Policies   (omit if no official-identity signal — say so)

| Identifier type | Applies to | Expires? | Validation | Policy source |
|-----------------|-----------|----------|------------|---------------|

Uniqueness across parties: [rule or "not required"]

## Roles

Representation: [plain / fixed / catalog] — [reason]

| Role | Form | May be held by | Validity? | Constraints / requirements |
|------|------|----------------|-----------|----------------------------|

## Address Book   (omit if no contact-point signal — say so)

Placement: [inside party / separate aggregate]

| Owner (kind) | Address kind | Uses | Validity | Notes |
|--------------|--------------|------|----------|-------|

Rules: [list, each tagged (R)/(A)/(I)/(X)]

## Relationships   (omit if no network signal — say so)

Placement: [inside party / autonomous]. Consistency: [immediate / eventual + reconciliation]

| Name | From (kind, role) | To (kind, role) | Validity | Rules |
|------|-------------------|-----------------|----------|-------|

Instances: [which concrete parties hold which relationships]

## Capabilities & Role Requirements   (omit if no capability signal — say so)

Placement: [inside party / own aggregate]
Capability type: [name] — mandatory scopes: [...]

| Party | Capability type | Scopes | Validity |
|-------|-----------------|--------|----------|

Role type: [name] requires [...]   (or "none")
Eligibility enforcement: [gate / advisory / both]

## Boundaries & Integration

| Aggregate | Contents | Own version / events |
|-----------|----------|----------------------|

| Consumer | Asks Party for | Owns itself | Point-in-time? |
|----------|----------------|-------------|----------------|

Graph questions: [list or "deliberately not modeled"]

## Worked Queries
[3–8 questions, each with the answer and the concept that decides it]

## Implementation Notes
[Key decisions, assumptions for unanswered questions, skipped layers, policies supplied from outside]
```

---

## Common Patterns & Pitfalls

### Pattern: Data Is Dumb but Stable; Rules Are Smart but Changeable

Which identifier a party may hold, which role a kind may take, whether an address may be added, whether a relationship may be created — each is a **defining policy** supplied to the model and enforced by the model. The check is stable logic; the concrete rules are its closure, chosen by a factory, configuration or tenant. Constant checks are enforced by the model, never only by an application service.

### Pattern: Heart, Not Pack Ox

*"Not every piece of information about a party must be in the party. Not every operation must be atomically synchronized with it. Most often eventual consistency is enough."* Run the five boundary questions of Step 7 on addresses, preferences, consents, relationships, capabilities. *"Boundaries, well designed, do not constrain. They build strength."*

### Pitfalls — the Kraken checklist

- **Semantic number as key** — breaks in the next market, on the next reassigned phone number, on the first shared mailbox
- **Flags, nulls and "theoretically"** — `isCustomer` + `isPartner` + `isVendor`, empty company name on people, and "theoretically a user can be a company": the user class with 120 fields and a prepaid balance
- **One class per role, or the Cartesian product** — three records for one company, copied on each transition, then `CustomerPartner` to patch it; nobody can answer "how many customers do we have?"
- **Hierarchy as parent pointer** — cannot link siblings or a person working for two organizations
- **Modeling responsibilities** — duties and their conditions of satisfaction belong to contract modules; model capabilities
- **Party as a shared library** — binary coupling, twelve slightly different copies, no place for history, audit or retention

---

## Quality Checks

Before returning the model, verify:

- [ ] Complexity level is stated, and every skipped layer is marked *deliberately not modeled*
- [ ] One party identifier space, no semantic key; each kind holds only its own data
- [ ] Every registered identifier type states applicability, expiry and where its policy comes from; no identifier and role gate each other circularly
- [ ] Role representation strategy is chosen with a reason; every role states which kinds may hold it
- [ ] Every address states kind, uses and validity; address-book rules are rules over the set
- [ ] Every relationship has a name, a direction, a role on each side and a validity; hierarchy is expressed as relationships
- [ ] Placement is stated with a reason for the address book, relationships and capabilities
- [ ] Consistency mode for every cross-party rule is stated (immediate or eventual with reconciliation)
- [ ] Every capability type lists its mandatory scopes and every capability has them plus a validity; role requirements are declarative with a "what is missing" answer; availability is routed out
- [ ] Aggregates reference each other only by party id
- [ ] Point-in-time consumers (orders, contracts) have a snapshot rule
- [ ] Concept mapping is complete; Unmapped Concepts is present (even if empty)
- [ ] Every worked query is traceable to a scope, role, relationship or validity value in the tables
- [ ] No (X) decision remains unconfirmed (Step 11.5); every clarifying answer or assumption is reflected in the model

---

## Example

**Input:** "A chain of medical clinics. The company MedicoLab runs facilities in Mokotów and Ursynów. Sonographers work at facilities — some are employees, some contractors. Patients register at reception, often without an online account; some corporate customers buy occupational-health packages for their staff. Staff hold licences that expire. Invoices go to the paying company's registered office; patients get results by e-mail; each facility publishes one contact address. Scheduling must know who can perform a prenatal ultrasound at Mokotów on a weekday morning and how many patients a person can take per day — booking itself stays in the scheduling system. The Senior Sonographer role may only be given to staff who can perform prenatal scans."

**Detected level:** 6 — operational capability (layers 1–6 signalled; layer 7 not needed).

**Output:**

```markdown
# Party Archetype Model: Clinic Network

## Complexity Level
Level 6 — operational capability. Layers modeled: identity & classification, official identity,
contact points, network, governed roles, capability. Deliberately not modeled: organizational graph.

## Concept Mapping

| Domain Concept | Party Archetype | Notes |
|----------------|-----------------|-------|
| MedicoLab | Party — Organization (company) | Employer, package provider |
| Facility Mokotów, Facility Ursynów | Party — Organization (unit) | Linked to MedicoLab by "organizational membership" |
| Sonographer, patient | Global roles | Classification; patients are persons only |
| Employee / contractor | Roles in relationships "employment" / "contracting" | Direction: person → company |
| Patient without online account | Party (person) with role patient | No credentials — identity ≠ account |
| Corporate customer buying a package | Party (company) with role customer + relationship "occupational-health package" | Its staff are the patients |
| Licence that expires | Registered identifier (licence number) + capability validity | Number is identity; what it permits is capability |
| Who can perform prenatal at Mokotów, weekday morning, max N/day | Capability "Medical Imaging" with protocol, location, temporal, quantity scopes | Possibility, not availability |
| Senior Sonographer eligibility | Role type with role requirements | Checked against capabilities |
| Registered office, result e-mails, facility contact address | Address book | Separate aggregate |

## Unmapped Concepts
- Appointment availability and booking — scheduling system (availability, not capability)
- Patient medical records — clinical records module

## Clarifying Questions & Answers
- Units vs companies? → both needed (facilities are units)
- Role representation? → governed catalog (Senior Sonographer has requirements)
- Contact points? → separate address book
- Address-book rules? → one active contact address per facility; billing use for organizations only
- Identifier uniqueness across parties? → national id and licence number unique system-wide
- Relationships? → autonomous; consistency → eventual with nightly reconciliation
- Capability placement? → own aggregate (licences renew independently)
- Role validity? → (assumed) none; "who was a sonographer in March" not required
- Re-registration → (assumed) no-op, recorded as such
- Point-in-time data? → invoices snapshot payer and patient data at visit time

## Party Kinds & Identity

| Kind | Own data | Examples |
|------|----------|----------|
| Person | first name, last name | Maria Nowak, Ewa Lis, patient Jan Kowalski |
| Organization — company | legal name | MedicoLab sp. z o.o., Zeta Software (corporate customer) |
| Organization — unit | name | Facility Mokotów, Facility Ursynów |

Identifier: one party id space for all kinds; random UUID generated in the application.

## Registered Identifiers & Policies

| Identifier type | Applies to | Expires? | Validation | Policy source |
|-----------------|-----------|----------|------------|---------------|
| National identification number | persons only | never | format + checksum | composed policy, hard-wired |
| Tax number | persons and organizations | never | format + checksum | composed policy, hard-wired |
| Medical licence number | persons only | yes — licence period | registry format | composed policy, hard-wired |

Uniqueness across parties: national id and licence number unique system-wide, enforced at write (A).
Applicability is keyed on party kind only; the licence gates the sonographer role, so keying the licence on the role would be circular.

## Roles

Representation: governed catalog — Senior Sonographer carries requirements; roles change without deployment.

| Role | Form | May be held by | Validity? | Constraints / requirements |
|------|------|----------------|-----------|----------------------------|
| patient | global | person | no | — |
| customer | global | company | no | — |
| sonographer | global | person | no | must hold a valid licence number |
| senior sonographer | global (role type) | person | no | Medical Imaging capability, valid, protocol ⊇ {prenatal}, quantity ≥ 10 per day |
| employee, contractor, member unit, patient, customer (from-sides); employer, principal, parent organization, care provider, provider (to-sides) | in relationship | see Relationships | via relationship | patient and customer sides require the global role of the same name |

## Address Book

Placement: separate aggregate keyed by party id.

| Owner (kind) | Address kind | Uses | Validity | Notes |
|--------------|--------------|------|----------|-------|
| MedicoLab (company) | postal | billing, mailing | open | registered office |
| Zeta Software (company) | postal | billing | open | occupational-health invoices go here |
| Facility Mokotów (unit) | postal | contact | open | published address |
| Facility Ursynów (unit) | postal | contact | open | published address |
| Jan Kowalski (person) | e-mail | contact | open | results delivery |
| Maria Nowak (person) | e-mail | contact | open | |

Rules: no two simultaneously valid addresses of the same kind and use for one party (I, Step 7);
billing use only for organizations (A); exactly one active contact address per facility (R).

## Relationships

Placement: autonomous. Consistency: eventual — nightly reconciliation flags violations.

| Name | From (kind, role) | To (kind, role) | Validity | Rules |
|------|-------------------|-----------------|----------|-------|
| employment | person, employee | company, employer | from hire date, open | only organizations employ; only persons are employees |
| contracting | person, contractor | company, principal | contract dates | as above |
| organizational membership | unit, member unit | company, parent organization | open | a unit belongs to exactly one company |
| patient registration | person, patient | unit, care provider | from first visit, open | from-side must hold the global patient role |
| occupational-health package | company, customer | company, provider | contract dates | from-side must hold the global customer role |

Instances: Maria Nowak — employment with MedicoLab; Ewa Lis — contracting with MedicoLab; both facilities — member units of MedicoLab; Jan Kowalski — registered at Facility Mokotów; Zeta Software — package with MedicoLab.

## Capabilities & Role Requirements

Placement: own aggregate per capability.
Capability type: Medical Imaging — mandatory scopes: protocol, location, temporal, quantity.

| Party | Capability type | Scopes | Validity |
|-------|-----------------|--------|----------|
| Maria Nowak | Medical Imaging | protocol: ultrasound {abdominal, cardiac, prenatal, thyroid}; location: {Mokotów}; temporal: Mon–Fri 08:00–16:00; quantity: max 15 per day | 2024-01-01 → 2027-01-01 (end exclusive) |
| Ewa Lis | Medical Imaging | protocol: ultrasound {abdominal, thyroid}; location: {Mokotów, Ursynów}; temporal: Mon–Sat 08:00–14:00; quantity: max 12 per day | 2025-01-01 → 2027-07-01 (end exclusive) |

Role type: Senior Sonographer requires Medical Imaging with protocol ⊇ {prenatal} and quantity ≥ 10 per day.
Eligibility enforcement: gate at assignment; reconciliation re-checks when a capability expires.

## Boundaries & Integration

| Aggregate | Contents | Own version / events |
|-----------|----------|----------------------|
| Party | id, identifiers, roles, names | yes |
| Address book (per party) | all addresses of one party | yes |
| Relationship (each) | name, two sides, validity | yes |
| Capability (each) | type, scopes, validity | yes |

| Consumer | Asks Party for | Owns itself | Point-in-time? |
|----------|----------------|-------------|----------------|
| Scheduling | who can perform procedure P at facility F on day D | availability, bookings | no |
| Invoicing | payer identity and the payer's billing address; patient identity | invoices | yes — snapshot at visit |
| Access management | does party X hold role Y | credentials, sessions | no |
| HR | employment relationships, licence validity | contracts, payroll | no |

Graph questions: deliberately not modeled.

## Worked Queries (evaluated on Wednesday 2026-09-09)
1. Can Maria perform a prenatal scan at Mokotów on Wednesday 10:00? Yes — protocol includes prenatal, location includes Mokotów, Wed 10:00 is inside Mon–Fri 08:00–16:00, 2026-09-09 is before 2027-01-01.
2. Is Maria free on Wednesday 10:00? Refused — availability belongs to scheduling; Party only says she *can*.
3. Can Maria perform a prenatal scan at Ursynów? No — location scope is {Mokotów}.
4. Can Maria take 18 patients on Thursday? No — quantity scope is 15 per day; the 16th is refused.
5. Can Maria scan on Saturday 10:00? No — temporal scope is Mon–Fri.
6. Can Ewa perform a prenatal scan at Ursynów on Saturday 10:00? No — location and time fit, protocol lacks prenatal.
7. Who can perform an abdominal scan at Ursynów on Saturday 10:00? Ewa only — Maria fails on location and day.
8. Who may hold Senior Sonographer? Maria — prenatal present, 15 ≥ 10. Ewa — not eligible; missing: protocol prenatal (quantity 12 ≥ 10 passes). On 2027-02-01 Maria's capability has expired (end 2027-01-01 exclusive), so she is no longer eligible and reconciliation flags her assignment; Ewa is still valid until 2027-07-01 but still lacks prenatal.

## Implementation Notes
- A company buying an occupational-health package is a customer, not a patient; its employees are the patients (package relationship plus individual patient registrations)
- Work location is expressed by the capability's location scope, not by a "works at" relationship — one source of truth for "who works where"
- Role assignments carry no validity (assumed); revisit if history of roles is required
- Duplicate role / identifier / address registrations are no-ops, recorded as such
- Identifier, role, relationship and address policies are hard-wired; no tenant variability expected
- Eventual consistency accepted for "unit belongs to exactly one company" and "one active contact address"; reconciliation runs nightly
- Organizational graph queries deliberately not modeled; the two-level hierarchy is answered by direct relationship queries
```

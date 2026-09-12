---
name: roles-archetype-mapper
description: Transform domain requirements into a Roles Archetype model. Identifies the subjects that play roles, the form of each role (classification, side of a relationship, part in a process), the representation strategy (plain name, fixed set, governed catalog), applicability and role-set policies, role scopes with override rules, capability-based eligibility, and the lifecycle, time and authorization boundaries. Adopts only the layers the domain signals.
argument-hint: "[domain requirements or feature description]"
---

# Roles Archetype Mapper

Transform any domain description in which the same subject *counts as* different things — a customer who is also a supplier, a company that orders but does not pay, a team that may or may not qualify as a delivery team — into a Roles Archetype model. The archetype's one inversion: **roles are data attached to a stable identity, never types, flags or columns.** *"Roles do not define parties; parties possess roles."*

**Output goal**: An implementable model that says, for every role, who may hold it, in which form and scope, how it is represented, how it is granted and withdrawn, whether holding it must be earned, and which questions other modules may ask about it — adopted only to the depth the domain needs.

## When to Use

**Use this skill when:**
- The same real-world subject appears under several names (user, customer, partner, vendor, employee, lead) or changes name over time
- Flags such as `isCustomer` / `isPartner` are accumulating, or one class per role has produced *"three records for the same company"*
- A process has several participants each with a distinct part: who submits, who pays, who receives, who executes
- Some subject kinds may not hold some roles ("only organizations employ"; "a department cannot be an employee")
- Holding a role must be earned — a certificate, a skill level, a covered region — not merely granted
- Roles must be added, changed or retired without a release, or a misspelled role name has legal consequences
- Notifications, invoices or personal data reach the wrong participant because one identifier stands for everyone involved ("the reminder went to the buyer, not the person having the treatment")

**Output is useful for:**
- Domain modeling sessions before implementation, and for deciding what the Party or Ordering models should delegate to a shared role model

## When NOT to Use — Fit Test

Before starting the mapping, apply this test. If the domain fails it, **stop and tell the user** that the Roles archetype does not fit, and briefly explain why.

### The core question

> *"Can I ask 'what does subject S count as here — on its own, toward whom, in which process — and may S hold that?' and does the answer change over time without a release?"*

If **yes** → Roles likely fits.
If every subject is exactly one kind of thing with exactly one part to play, forever → a fixed field is adequate. Do not map.

**Counter-questions that route elsewhere:**
- *"Who is S at all — person or organization, which identifiers, which addresses?"* → party archetype. Roles say what S *counts as*; Party says who S *is*
- *"What must the role holder do, by when, to what standard?"* → contracts, product or plan-vs-execution. The sources survey responsibilities and decline them: *"responsibilities stay in the books, and an empty role structure stays in the systems"*
- *"Who may log in, with which permissions on which screen?"* → identity & access management. This archetype supplies the fact *"S holds role R"*; it never decides a permission
- *"Can S do X at location L on day D, how many times?"* → capabilities (party archetype). Roles *consume* capabilities as eligibility; they do not describe ability
- *"What was ordered, at what price, in what state is it?"* → ordering. Roles supply the participation vocabulary; the order owns the process
- *"How much X does S have?"* → accounting

### Signal table

| Signal in requirements | Likely archetype fit? |
|------------------------|-----------------------|
| "the same company is our supplier and our customer" / "an employee who also buys from us" | ✅ Yes |
| "who pays is not who orders"; "someone submits, someone pays, someone receives, someone executes" | ✅ Yes — participation roles |
| "the confirmation goes to whoever clicked buy"; "the invoice went to the wrong company"; "the salon saw the buyer's data, not the recipient's" | ✅ Yes — participants collapsed into one field |
| `isCustomer`, `isPartner`, `isVendor` flags; a `user_type` column that keeps growing | ✅ Yes — flags in disguise |
| "a lead became a customer, then a partner — why do we have three records?" | ✅ Yes — role change over time |
| "only companies can be employers"; "a team cannot be a patient" | ✅ Yes — applicability |
| "to execute medical deliveries a carrier needs ISO 13485 and must cover the region" | ✅ Yes — eligibility |
| "new partner categories appear every quarter"; "a typo in a role name ends in a report to the regulator" | ✅ Yes — governed roles (catalog or fixed set, Step 5) |
| "every user is one person with one login and one fixed function" | ❌ No — a plain user model |
| "password, permissions per screen, sessions, tokens" | ❌ No — identity & access management |
| "the warehouse manager must keep 95% of items in stock over 30 days" | ❌ No — obligations belong to contracts or plan-vs-execution |
| "is the doctor free on Wednesday at 10:00" | ❌ No — availability, not role |
| "user type: admin / regular" | ⚠️ Borderline — a permission bundle (IAM) or a business classification (role)? Ask which *business* rule reads it |
| "job title" | ⚠️ Borderline — a classification of the person, or a side of an employment relationship? It is the second when the employer matters |

### Borderline cases — how to decide

- **Customer vs user.** *"Is a customer always a user? … A user without a user? This model doesn't add up."* They are two facts about possibly two subjects. A customer may have no account; an account may hold no business role. Both are roles; neither is the identity.
- **The same word in two forms.** "Supplier" can classify a party on its own or be one side of a supply relationship. Model the classification when every relationship radiates from your own organization; model the relationship side when the counterparty and direction matter. When both exist, say which one answers "is S a supplier?" (Step 4).
- **A setting dressed as a role.** "Roaming enabled", "newsletter subscriber" — if no business rule ever asks *who counts as this*, it is a preference, not a role. A role earns its place by being read by a policy, a query or a process (a heuristic of this skill; the sources model such names as roles without comment).
- **Guest participants.** A parcel dropped at a locker has a sender with no account. That is still a participation role — held by a snapshot of a subject, not by a registered identity (Step 7). Allowed only as a conscious business decision.

### If the domain does not fit

Output:

```
## Archetype Fit Assessment: ❌ Does Not Fit

The Roles archetype requires subjects that count as different things — on their own, toward
other subjects, or inside a process — and whose roles change without redeploying. This domain is
a [single-kind user model / access-control model / obligations model / ...] because:
- [specific reason from the requirements]
- The natural question is "[who may log in / what must be done / ...]", not "what does S count as here, and may it?"
```

Do NOT suggest alternative patterns or architectures. Stop here.

---

## Mapping Workflow

### Step 0: Get Requirements

- If provided as argument, use it directly
- If not provided, scan the recent conversation for domain context. If found, use that.
- Only if no argument AND no context in session, ask:
  > "Describe the domain — who takes part, what each of them counts as in which situation, what may hold which role, and what the system must be able to say about it?"

---

### Step 1: Assess Complexity Level

Identify **every layer that has a signal**. Level 1 is the foundation; levels 2–5 are independent of each other above it — a domain may need participation roles without relationships, or a catalog without eligibility. Model each layer with a signal, **and no further**. Report the highest level modeled, but adopt layers by signal, not by number.

| Level | Name | Signal in requirements | Naive solution that breaks |
|-------|------|------------------------|----------------------------|
| 0 | **Single fixed role** | One kind of subject, one function, forever | (none — a field is adequate) |
| 1 | **Classification roles** | Subjects hold sets of named roles, gained and dropped over time; some kinds may not hold some roles | `isCustomer` flags; one class or table per role; nulls as a modeling device |
| 2 | **Relationship roles** | A role means something only *toward* a counterparty: employer/employee, manager/subordinate, holder/provider; direction and rules over the pair matter | parent pointers; a flat link table with no meaning |
| 3 | **Participation roles** | Several participants in one process or contract; some roles fixed for the whole, others per part (line, shipment, visit); participants unregistered or with changing data | one `userId`; a column per role; "add more fields" |
| 4 | **Governed roles** | Roles carry meaning beyond a name — description, constraints, traits shared by all holders; or a name error has legal or financial consequences; or roles must change without a release; multi-tenant | free-text names drifting ("Client", "client", "klient"); a growing set of flags |
| 5 | **Eligibility** | Holding a role must be *earned*: level, certificate, protocol, region, capacity; "what is missing" must be explainable; expiry withdraws eligibility | tags, text fields, *"hundreds of ifs"* |

**Guidance — which steps to run:**
- Level 0: the archetype is overkill. Emit the ❌ Does Not Fit block and stop; do not produce a model.
- Always: Steps 0–4, Step 5 (representation), Step 6 (policies), Steps 9–9.5.
- Add per signalled layer: level 2 → the relationship part of Step 4 and the relationship rules of Step 6; level 3 → the role-set policy of Step 6 and Step 7; level 4 → Step 5 chooses between a fixed set and a catalog (below it, plain names); level 5 → Step 8. At level 1 only the applicability half of Step 6 runs.

Partial adoption is normal — *"archetypes are a way of thinking; choose the elements that fit your business."* Mark skipped layers in Implementation Notes as *deliberately not modeled*.

**Handoff to siblings.** If the domain is an order-shaped process, run `ordering-archetype-mapper` for the process itself and use this skill for the role vocabulary, scopes and eligibility it consumes. If the domain is mainly about who the subjects *are*, run `party-archetype-mapper` for identity and let this skill own the role layer (Steps 4–8). State in Implementation Notes which model owns each role fact, so no two models define it.

---

### Step 2: Ask Clarifying Questions

Before continuing, identify gaps between the requirements and the archetype. Ask about **two categories** of questions in a single `AskUserQuestion` call (up to 4 questions per call; split into multiple calls if more needed). Ask only about layers Step 1 selected, and only the questions whose answer changes the **shape** of the model. Take the simplest option for the rest and mark it *(assumed)*.

#### Category A — Standard role decisions

Frame as design choices, not assumed defaults:

- **Role forms**: Classification only / sides of relationships / parts in a process / a mix? (Step 4)
- **Representation**: Plain names / a fixed set in code / a governed catalog? (Step 5)
- **Same name, two forms**: When "customer" is both a classification and a relationship side, which one answers "is S a customer?"
- **Validity on assignments**: None (a role is held or not) / a start and end with history ("who *was* a manager in March")?
- **Eligibility enforcement** (level 5): Refuse the assignment / refuse at process confirmation / advisory query only?
- **Participant identity** (level 3): Reference by identifier / immutable snapshot of the data needed for fulfillment?
- **Supersession**: Does a promotion keep the old role (customer *and* VIP) or replace it (manager → director)?

#### Category B — Gap-triggered questions

Scan the requirements for **anything the archetype supports but the requirements do not mention**. Ask whether each dimension is wanted. Reason freely; examples:

- **Applicability**: Which subject kinds may hold each role? Does holding one role require or forbid another? Does it require an identifier (a tax number for a supplier)?
- **Cardinality per scope** (level 3): Exactly one payer, or two? May a line have its own receiver? Which roles are forbidden per line?
- **Unregistered participants** (level 3): May someone take part without being a registered subject? With which restrictions (no history, no aggregation)?
- **Type constraints and extensions** (level 4): Conditions on the role type itself ("doctor: at least 35 years old"); things attached once to the type for all holders (a reporting duty)?
- **Per-tenant catalogs** (level 4): Do role definitions differ per tenant or region? (The sources name multi-tenancy as a reason to choose a catalog but never model a tenant-scoped definition.)
- **Re-check on expiry** (level 5): When a certificate lapses, is the role withdrawn, flagged, or left until the next check?
- **Consumers**: Which modules will ask "does S hold R?" Are any authorizations whole approval chains derived from relationships rather than one role?
- **Outside the sources** — ask, but say the archetype's sources are silent: mutual exclusion / separation of duties; role hierarchies (does "senior officer" imply "officer"?); delegation and acting on behalf of.

Collect answers before proceeding. If the user cannot answer, document the assumption in **Implementation Notes**.

#### Handling "it depends / both / varies by situation" answers

Always include **"To zależy / It depends"** as an explicit option in every `AskUserQuestion` call, as the last option. If selected, treat it as a **variable policy**:

- Name the *policy* the model will accept (applicability policy, role-set policy, eligibility requirements provider, validity rule)
- Note in **Implementation Notes** that the policy is chosen outside the role model — by tenant, region, context — and passed in. *"Data are stupid but stable; rules are smart but changeable."*
- Do **not** model the decision logic inside the role model

#### If `AskUserQuestion` is not available

Do not stop. Use these defaults, list each in **Clarifying Questions & Answers** marked *(assumed)*, and treat every Category B gap as *not wanted* unless the requirements imply it: plain names unless a catalog signal exists; the classification form answers "is S an R"; no validity on assignments; promotion is additive; eligibility refused at assignment for classification roles and at confirmation for participation roles; snapshot when participants are unregistered or their data changes, else reference; every role allowed for every subject kind unless the requirements say otherwise; for a process, the sources' example roster — exactly one initiator, exactly one executor, at most one payer, any number of recipients, whole-process roles forbidden on a part — each marked (X — confirm); removal not policed; no re-check on expiry; single-subject rules immediate, two-sided rules reconciled.

---

### Step 3: Map Domain Concepts to Roles Archetypes

For each significant noun and verb in the requirements, produce an explicit mapping table:

```
| Domain Concept       | Roles Archetype | Notes |
|----------------------|-----------------|-------|
| [domain noun/verb]   | Subject / Role (classification) / Role (relationship side) / Role (participation) / Role Type / Applicability Policy / Role-Set Policy / Scope / Snapshot / Requirement / Consumer query | [why] |
```

An adjective-qualified noun ("corporate customer", "premium seller", "certified carrier") is almost always a role or a role with requirements on an existing subject, never a new subject kind. A noun that names *what someone can do* (ultrasound, ADR transport) is a capability, not a role — map it as an input to Step 8. A constraint on who the holder *is* (kind, age, identifier, roles held) is applicability (Step 6); a constraint on what the holder *can do* (level, certificate, coverage) is eligibility (Step 8).

After the table, list any domain concepts that **could not be mapped**:

```
## Unmapped Concepts

The following domain concepts have no clear roles archetype equivalent:
- [concept] — [reason it doesn't fit / decision needed]
```

This section must be present even if empty (`None identified`).

---

### Step 4: Identify Subjects and Role Forms

**Subjects** are whatever holds roles: persons, organizations, organizational units, teams. A subject exists before it plays any part, and the same subject holds zero or more roles at once. *"Person and Organization are always the same. It is roles and relationships that give them context."* Do not model the subjects themselves here — the party archetype owns identity; record only which kinds exist and which may hold roles.

**Three forms of role — decide the form of every role:**

| Form | Meaning | Answers | Use when |
|------|---------|---------|----------|
| Classification | What S counts as on its own: customer, supplier, VIP, cost centre | "is S an R?" / "all R's" | The only relationships that matter radiate from your organization outward. *"Simpler, faster and precise enough for most cases"* |
| Relationship side | What S is *toward* T under a named, directed link: employee↔employer, account holder↔provider, manager↔subordinate | "who is S to T?" / "who employs S?" | Links between third parties carry meaning; direction says who initiates, who bears the effect. Level 2 |
| Participation | What S does *inside one process or contract*: orderer, payer, receiver, executor, delivery contact | "who pays for this order?" / "which roles apply to this line?" | Several participants per process, varying in number and combination between instances. Level 3 |

**The over-modeling test:** a manufacturer with 1,000 employees, 30 partners and 18 suppliers could create one relationship from itself to every one of them — *"formally correct; business-wise, like putting your pants on over your head."* Classification suffices when nobody will ever query by counterparty.

**Rules:**
- **Roles are data, not types.** The same subject gains and loses roles without migration, inheritance or duplicated records
- **A role is a trait at a moment**, added and removed explicitly, each change observable as an event (Step 9)
- **Names are domain vocabulary**, and one subject may carry names from several vocabularies at once (account holder, loan applicant, card holder)
- **A role named with a counterparty** — manager *of* the branch, advisor *of* this customer — is a relationship side; a role the domain uses without one is a classification. Two names for one fact ("customer" and "account holder" when they never differ) are one role; record the alias in the concept mapping
- **Role membership is a set**: no multiplicity, no primary role, nothing attached to one assignment. "Supplier *of category X*" is a capability scope or a relationship, not a role variant
- **Relationship sides are (subject, role) pairs**; the same two subjects may be linked many times under different role pairs (holder/provider, borrower/lender). Symmetric links are weak operationally — build them from two directed ones. Organizational structure (a unit belongs to a parent) is itself such a relationship — hierarchy emerges from role-typed links, *"a flag or a parent id does not solve this"*; model its sides here only when a rule reads them (an approval chain does)

**This is a design decision — the same name in two forms.** A global "patient" classification and a "patient" side of a care relationship may coexist. Choose: the classification answers "is S a patient?" and the relationship side *requires* it (one source of truth); or the relationship side alone defines it (no classification). The reference implementation leaves the two disconnected; decide which form is the source of truth and make the other require it (the party mapper defaults to the classification). Two sources of one fact will drift.

**Records into:** the Subjects & Role Forms section.

---

### Step 5: Choose the Representation

*Runs fully only if the governed-roles layer (4) was adopted in Step 1; below it, plain names.*

**This is a design decision — how a role is represented** (absent a signal, plain names):

| Strategy | Shape | Fits when | Breaks when |
|----------|-------|-----------|-------------|
| Plain name | An immutable name; assign = add to the subject's set after the policy check | MVP, simple systems, roles that are only a classification | Names drift; no description, no constraints, reporting pain — *"live fast, die young; running production in an Excel sheet"* |
| Fixed set in code | One type per role; each knows which subject kinds it serves; the policy is the type; traits and behaviour shared by all holders live on it | 2–3 stable roles; errors have financial or legal consequences; code must be self-describing and auditable | A new role is a release — *"agility disappears, concrete remains"* |
| Governed catalog | Role **types** (identity, name, description, constraints, extensions, requirements) and role **instances** that reference a type | Large, dynamic, multi-tenant; roles change without deployment; requirements must hang somewhere shared | Centralization: changing a type changes every instance at once |

The sources teach with plain names for clarity and are *"forced back"* to the catalog the moment a role needs requirements. Do not read plain names as the recommendation; read them as the floor. At level 4 choose: a fixed set when the roles are few and stable and a name error has legal consequences; a catalog when roles change without a release, differ per tenant or need requirements. Both signals at once → a catalog whose types carry the constraints.

**A role type is *"the label on the yoghurt"*:** a name, a description, and the conditions for using it. It carries:
- **Constraints** — conditions on the holder ("doctor: a person at least 35 years old"), checked when the role is assigned (the sources do not say how a constraint reads subject data)
- **Extensions** — anything shared by all holders, attached once to the type, not to hundreds of instances (a reporting duty, a default operating scope). This is the only place obligations survive in the archetype
- **Requirements** — at level 5, what a subject must be *able* to do to hold the role (Step 8)

**Rule:** anything shared by every holder attaches to the type; anything specific to one holder attaches to the instance. A type with no constraints and no requirements is open to anyone — and most roles are.

**Convention (catalog placement):** a catalog shipped as a shared library re-releases every consumer on each new role type — *"a shared kernel"*. Keep it behind a service contract.

**Records into:** the Representation & Catalog section.

---

### Step 6: Define Applicability and Role-Set Policies

Assigning a role is not a set insert. *"A role is not only a description; it is also a gate of control."*

**Applicability policy — may this subject hold this role?** One predicate over (subject, role), keyed on:
- **subject kind** — the most common: only organizations employ or supply; only persons are employees, patients, consumers; persons and companies may be customers, departments and teams may not
- **roles already held** — the sources use this only as a precondition elsewhere ("a customer number may be held by a person or a company, provided it holds the customer role"); dependencies ("no buyer without customer") and exclusions ("not both approver and requester") are expressible but outside the sources — mark them
- **other facts about the subject** — an identifier ("a supplier needs a tax number"), an attribute (age), or, at level 5, capabilities (Step 8)

**Rules:**
- **Enforced by the model, always.** *"If some element is a constant element of our logic, it should be enforced by the model"* — never only in an application service, where the next service will forget it. Every path that assigns a role runs the policy: direct assignment, registration with roles, and building a relationship side
- **Small policies compose.** Many single-rule policies that *abstain* on roles they do not recognize, combined with AND, beat one type switch with a hundred branches
- **Dynamic choice → policy factory.** When rules differ by tenant, region or context, the model accepts the policy; something above it chooses which. When they never differ, wire them statically — *"the choice of closure"*
- **Removal is a decision too.** State whether withdrawing a role runs a policy (a customer with open orders?) or is always allowed

**Role-set policy — does this combination of roles make sense here?** (Level 3, one policy per scope; also the relationship rules of level 2.) Express every role in a scope as one of four shapes:

| Shape | Meaning | Example |
|-------|---------|---------|
| Exactly one | Required, single holder | one orderer per order; one executor |
| At most one | Optional, single holder | payer; payment receiver |
| Any number | Optional, many holders | receivers; delivery contacts |
| Forbidden here | Never legal in this scope | orderer on a line; delivery contact on the whole order |

Counts are per role, not per subject — a subject holding three roles counts once toward each. When a shape depends on something outside the model (an amount threshold), record the parameter and pass it in. **Convention:** the policy runs when the scope is built, so an invalid scope is unrepresentable. Whether a violation report names the first offending role or accumulates all of them is an implementation choice.

**Relationship rules (level 2)** work at two levels: the *role* is constrained by subject kind (above), and the *relationship* is constrained — with the same four shapes, counted over one subject's relationships — **quantitatively** (at most 10 subordinates; at most 3 employers), **qualitatively** (only a company employs; a care relationship only if both sides hold the roles) and **temporally** (not terminable before three months; a mentorship of at most one year). A rule that reads *both* sides locks two subjects in one transaction — *"scalability, flexibility and availability are in practice more important than strict immediate consistency"*; most such violations may be repaired by a reconciliation job. State which rules are immediate and which are reconciled.

**Records into:** the Applicability & Role-Set Policies section.

---

### Step 7: Define Scopes and Participation

*Run only if the participation layer (3) was adopted in Step 1 — otherwise say so. If the process has no parts, fill the whole-process scope only and say so.*

A process (order, case, shipment, visit) is an operational contract between parties in roles: who initiates, who bears the cost, who receives the result, who executes, who accepts payment. *"Roles are conceptually stable, but their number in one instance varies, their combinations are dynamic, and we do not know all of them at design time."* Therefore: a **list of (participant, set of roles)**, never a column per role. Dedicated fields are acceptable only when the roster is small and identical in every instance — and only until the first new role.

**Scope — where does each role hold?**

| Scope | Roles that live there | Rule |
|-------|----------------------|------|
| Whole process | orderer, payer, executor, payment receiver — *"it rarely makes sense for each line to have a different orderer"* | Consistent for the whole; forbidden per part |
| Part of the process (line, shipment, visit) | receiver, delivery contact, installation recipient, pickup authorized | Local and optional; meaning appears only for a specific product or fulfillment |
| Both | a role with a default for the whole that may be refined per part — in the sources' example only the receiver | The scope overlap is what makes an override rule necessary; a domain may have several |

**Override rule — by role, not by participant.** A part with no roles of its own inherits everything from the whole; that is a legal, first-class state, not an error. A part that names role R overrides the whole's holder of R for that part only; every unnamed role still inherits; a part may add a role that never existed at the whole. Give the process **one query** that answers "which roles apply to this part?" — the merge logic must not be scattered as *"check here, and if not there, check somewhere else."*

**Participant identity — this is a design decision:**
- **Reference** (subject identifier + roles) suffices when every participant is registered and the data needed for fulfillment never changes or does not matter
- **Snapshot** — an immutable copy of the minimum data for fulfillment, contact, audit and complaints — when data changes after the fact (*"Ms Świątek orders this week and is Ms Świątek-Piątek next week; the order still concerns the first name"*) or when participants are unregistered (a parcel sender at a locker). One snapshot per participant, a set of roles pointing at it; the snapshot never follows the master record
- **Unregistered participants** are a conscious business decision: no identifier means history and cross-instance aggregation get harder and less trustworthy

**Where assignments come from** is a process decision the model must *enable*, not hide: the orderer usually arrives with the command or the security context; the executor may be chosen by the user, defaulted per product, region or channel, or routed by rules; the payment receiver is often a fixed unit. Name the source of every role; a role the policy requires but the requirements never mention is inferred and tagged (X — confirm).

**Records into:** the Scopes & Participation section.

---

### Step 8: Define Eligibility

*Run only if the eligibility layer (5) was adopted in Step 1 — otherwise say so.*

Instead of modeling what a holder *must do* (declined — see the fit test), model what a subject must be *able to do* to hold the role. *"A responsibility is an obligation; a capability is a possibility."* A role becomes *"a formal declaration"*: Backend Delivery Team *"does not say 'a team of backend people'. It says: a team that must possess the capability Backend Development with skills at a level of at least 8, with AWS and Kubernetes certificates, which operates according to the Agile Scrum protocol"* — and the capability must be active.

**Structure:**
- **Requirements hang on the role type**, shared by all holders — *"a contract template"* — never on individual assignments
- **Role requirements** = a list of **capability requirements**; every one must be met (AND). No trade-off: excellence in one clause never compensates for absence in another
- **Capability requirement** = a required capability type + zero or more **scope requirements** (where, when, how many, how well, by which rules, with what). Met by *any one* currently valid held capability of that type whose scopes meet every scope requirement (ANY across held, AND across scopes). No scope requirements → any valid capability of the type suffices. A requirement is met by **one** capability, never by the union of several: a subject holding the required type twice, each covering half the scope requirements, does not qualify — this decides how capabilities are granulated
- **Scope matching runs in one direction — what is held covers what is asked:** a superset for sets (certificates, regions, products, resources), ≥ for ranks and capacities, coverage for time windows. Holding more than required always passes
- **Expiry withdraws eligibility silently.** An expired capability does not count; no separate "expired" state is needed
- **"What is missing"** — the unsatisfied clauses — is as valuable as the yes/no. Report it, not just the verdict. The sources report the failing capability requirement; reporting the failing scope inside it is a reasonable generalization

Capabilities themselves belong to the party archetype; this step only *reads* them. **The process must not know what a certificate is:** an order receives the requirements for a role in *this* context from a requirements provider and enforces the pass/fail contract — it never interprets ISO 13485 or geography.

**This is a design decision — enforcement.** The party sources produce only a verdict — *"there are no ifs in a service, no validation in the UI; simply, the logical formula is not satisfied"* — and never say where it is enforced; the ordering sources place the check at submission or at confirmation (*"the order stops being a working intent and becomes a binding contract"*); the party reference implementation exposes eligibility as a query and never consults it when a role is assigned. Choose per role: refuse at assignment, refuse at process confirmation, advisory query only, or gate now and re-check by reconciliation when capabilities expire (the last is outside the sources — nothing in them reacts to expiry).

**Convention (matchmaking):** the same formula run over a population — "all carriers eligible for hazmat to Cracow" — is a search, not an assignment. Note whether the domain needs it.

**Records into:** the Eligibility section.

---

### Step 9: Define Lifecycle, Time and Boundaries

**Lifecycle.** Assigning and withdrawing a role are commands on the holder, each observable as an event — the narrative says only that the system *"reflects it in events"* and names none. **Convention (idempotence — from the reference implementation, not the narrative):** assigning a held role or withdrawing an absent one *succeeds* — the intent is already realized — but changes no state and publishes no external event; the skip is recorded internally with a machine-readable reason. If the domain needs a role history, say so here.

**Rules:**
- **Roles accumulate.** Customer → VIP → Premium VIP keeps all three unless the domain says otherwise. Supersession (manager → director drops manager) is an explicit two-step, or a single command when the domain names it — decide in Step 2
- **The subject must exist first.** Assigning a role to an unknown subject fails; it is not a registration path
- **Placement.** Roles are *"authorization-critical"*, so they live inside the identity aggregate and change atomically with it, protected by its version. Participation roles live inside the process aggregate. Relationship sides live with the relationship

**Time — this is a design decision.** The sources call a role *"a trait at a given moment"* but model validity for identifiers, addresses, relationships and capabilities — never for a role assignment — although the reference describes, and does not build, a role *instance* that would carry one. Choose: no validity (held or not; history only through events), or a validity on each assignment when the domain asks "who *was* an R in March" or must pre-date a role. Relationship sides inherit the relationship's validity; a 24-month subscription *"expires automatically, with no manual intervention."* Participation roles are frozen with the process — say whether they may change after confirmation (outside the sources).

**Authorization boundary.** Other modules ask *"does subject X hold role Y?"* — and get a fact. *"Instead of every application creating its own user and role tables, a process can simply ask."* Permissions, screens and credentials are decided by the consumer. Some authorizations are whole chains (approval above a threshold by the branch manager, above a higher one by the board) — derive them from relationships, not from one role check. Product and pricing modules ask *"may this partner sell this category?"* — a role-plus-capability question, answered here.

**Records into:** the Lifecycle, Time & Boundaries section.

---

### Step 9.5: Decision Sanity Check

**Before producing the final output**, enumerate every concrete decision embedded in the draft model and verify each one has a source:
- **(R)** — explicitly stated in the requirements
- **(A)** — asked and answered in Step 2
- **(I)** — inferred from a structural necessity of the archetype (state which)
- **(X)** — neither: assumed silently

| Decision area | Example decisions to check |
|---------------|---------------------------|
| Role form | Classification vs relationship side vs participation for each role; which form answers "is S an R?" |
| Representation | Plain / fixed / catalog; what the type carries; per-tenant catalogs |
| Applicability | Which kinds hold which roles; dependencies between roles; whether removal is policed |
| Role-set cardinalities | Exactly one / at most one / any / forbidden, per role per scope |
| Scope split | Which roles are whole-process, which per part, which both |
| Participant identity | Reference vs snapshot; unregistered participants allowed |
| Assignment source | Command, security context, default, routing, fixed unit — per role |
| Eligibility | Requirements per role type; enforcement moment; re-check on expiry; what-is-missing exposed |
| Time | Validity on assignments; changes after confirmation; supersession |
| Consistency | Which rules are immediate, which reconciled |
| Consumers | Who asks "does S hold R"; chains derived from relationships |

**For every (X) decision found:**
1. Low impact (technical, easily changed): mark as explicit assumption in Implementation Notes.
2. Affects business behavior (cardinalities, enforcement moment, snapshot vs reference, supersession): **stop and ask** using `AskUserQuestion` before delivering the model. Without `AskUserQuestion`, apply the Step 2 defaults and mark *(assumed)*.

Do not deliver the model until all material (X) decisions are either confirmed or documented as explicit assumptions.

---

## Output Format

```markdown
# Roles Archetype Model: [Domain Name]

## Complexity Level
[Level N — name]. Layers modeled: [...]. Deliberately not modeled: [...].

## Concept Mapping

| Domain Concept | Roles Archetype | Notes |
|----------------|-----------------|-------|
| ...            | ...             | ...   |

## Unmapped Concepts
[List or "None identified"]

## Clarifying Questions & Answers
[Question → answer, or "(assumed) default" — one line each]

## Subjects & Role Forms

| Subject kind | May hold roles? | Notes |
|--------------|-----------------|-------|

| Role | Form | Answers | Same name elsewhere? |
|------|------|---------|----------------------|
| [name] | classification / relationship side (of [relationship], toward [role]) / participation (in [process]) | [question] | [which form is the source of truth] |

## Representation & Catalog
Strategy: [plain name / fixed set / governed catalog] — [why].
[If fixed set or catalog — one row per role type:]
| Role type | Description | Constraints | Extensions | Requirements (see Eligibility) |
|-----------|-------------|-------------|------------|--------------------------------|

## Applicability & Role-Set Policies

| Role | May be held by | Requires / forbids | Checked at | Removal policed? |
|------|----------------|--------------------|------------|------------------|

[If the participation layer is adopted — role-set policy per scope:]
| Scope | Role | Shape (exactly one / at most one / any / forbidden) | Source |
|-------|------|------------------------------------------------------|--------|

[If the relationship layer is adopted — relationship rules:]
| Relationship | Sides (role → role) | Quantitative | Qualitative | Temporal | Immediate or reconciled |
|--------------|---------------------|--------------|-------------|----------|-------------------------|

## Scopes & Participation   (omit unless the participation layer is adopted — say so)
Process: [name]. Parts: [name].
Participant identity: [reference / snapshot — fields: ...]. Unregistered participants: [yes/no + restrictions].
Effective-roles query: [where it lives; inherit / override / add rule].
| Role | Scope | Source of assignment | Source tag (R/A/I/X — Step 9.5) |
|------|-------|----------------------|-----|

## Eligibility   (omit unless the eligibility layer is adopted — say so)
Enforcement: [at assignment / at confirmation / advisory / gate + reconcile]. Requirements provider: [who supplies them per context].
| Role type | Capability required | Scope requirements | Match rule (any held capability × all scope requirements) | What-is-missing exposed? |
|-----------|---------------------|--------------------|-----------|--------------------------|
[Worked eligibility check: one eligible subject, one not, with the missing clauses]

## Lifecycle, Time & Boundaries
Events: [assigned / withdrawn / skipped(reason)]. Supersession: [additive / replace]. Placement: [aggregate per form].
Validity on assignments: [none / from–to]. Changes after confirmation: [...].
Consumers: [module → question asked]. Chains derived from relationships: [...]. Consistency: [which rules are immediate, which reconciled].

## Implementation Notes
[Key decisions; (assumed) defaults; deliberately not modeled layers; which sibling model owns each shared role fact; outside the sources: [mutual exclusion, hierarchy, delegation… if asked]; decision provenance: every material decision with its (R)/(A)/(I)/(X) tag]
```

---

## Common Patterns & Pitfalls

### Pattern: The Role Is a Fact; the Permission Is Someone Else's Decision

*"Does X hold role Y?"* is answered here. What Y lets X see or click is decided by the consuming system. Keeping the two apart lets a role change without a redeploy of every screen, and lets a screen change without touching the role catalog. When a requirement reads "admins can delete", ask which *business* fact makes someone an admin — that is the role; the deletion right is the consumer's.

### Pattern: Variable Rules Are Chosen Above the Model and Passed In

Applicability, role-set cardinalities, eligibility requirements and validity all vary by tenant, region or context in real systems. The role model *accepts* a policy and enforces it mechanically; a factory or configuration above it chooses which. *"Data are stupid but stable; rules are smart but changeable."* Model the parameter, not the decision logic.

### Pitfall: A Flag Wearing a Role's Name

"Roaming enabled", "newsletter subscriber", "beta tester" pass the syntax of a role and fail its purpose. If no policy, query or process ever asks *who counts as this*, it is a setting. Conversely, an "employee" that no flag ever recorded — *"so every system recognizes him by his role"* — was a role all along.

### Pitfall: Two Sources of One Fact

A classification "patient" and a relationship side "patient" that are maintained independently will drift; the reference implementation lets relationship sides be labelled with roles the subject never held. Decide which form is the source of truth and make the other require it; the party mapper defaults to the classification.

### Pitfall: Requirements as a Query Nobody Runs

A catalog of role requirements that assignment never consults documents intent without enforcing it. If eligibility is advisory, say so and name who reads the answer; if it gates, name the moment.

### Pitfall: Scattered Effective-Roles Logic

Three call sites each doing "look at the line, else the order, else the default" is the participation-role equivalent of the flag model. One query, one merge rule, by role.

---

## Quality Checks

Before returning the model, verify:

- [ ] Every role has exactly one declared form, and a same-name role in two forms names its source of truth
- [ ] The representation strategy is justified by a signal, and a fixed set or catalog appears only where the governed-roles layer was adopted
- [ ] Every role has an applicability rule (even "any subject kind") and a statement of where it is enforced — by the model, on every assignment path
- [ ] If the participation layer is adopted, every participation role has a shape in every scope, and the override rule is by role with one effective-roles query
- [ ] Participant identity (reference / snapshot) is decided, and unregistered participants are an explicit decision
- [ ] If the eligibility layer is adopted, requirements hang on role types, the enforcement moment is named, and the worked check shows what is missing
- [ ] Lifecycle events, idempotence and supersession are stated; validity on assignments is a recorded decision
- [ ] The authorization boundary names consumers and their questions; no permission or credential appears in the model
- [ ] Obligations, availability and identity data are routed to their archetypes, not modeled here
- [ ] Concept mapping is complete; Unmapped Concepts is present (even if empty)
- [ ] All clarifying answers or (assumed) defaults are reflected; every (X) decision is confirmed or documented
- [ ] Skipped layers are listed as deliberately not modeled

---

## Example

**Input:** "A medical logistics platform. Hospitals and clinics order deliveries of medical equipment. The ordering hospital is not always the payer — a hospital group often pays centrally — and a line can go to a different ward with its own contact person, who may not be in our system. Deliveries are executed by courier companies. Only a courier certified for medical devices (ISO 13485) with cold-chain handling and coverage of the destination region may execute. Courier companies are also our customers when they buy packaging. New partner categories appear every quarter and must not wait for a release."

**Detected level:** 5 — layers 1 (classification), 3 (participation), 4 (governed roles, as a catalog), 5 (eligibility). Layer 2 (relationship roles) has no signal.

**Output:**

```markdown
# Roles Archetype Model: Medical Logistics Platform

## Complexity Level
Level 5. Layers modeled: classification roles, participation roles, governed roles (catalog), eligibility.
Deliberately not modeled: relationship roles (no party-to-party link carries meaning in the requirements).

## Concept Mapping

| Domain Concept | Roles Archetype | Notes |
|----------------|-----------------|-------|
| Hospital, clinic, hospital group, courier company | Subject (organization) | Identity owned by the party archetype |
| Ward | Subject (organizational unit) | May receive; may not order or pay |
| Ward contact person | Subject (person), possibly unregistered | Snapshot participant |
| "customer" (buys deliveries or packaging) | Role — classification | A courier company can hold it too |
| "courier partner" | Role — classification, with requirements | Governed type |
| Orders, pays, receives, executes, contact on site | Role — participation in a delivery order | Whole-order vs per-line scopes |
| ISO 13485, cold chain, region coverage | Requirement on the courier partner / executor type | Read from capabilities; not modeled here |
| "partner categories appear every quarter" | Governed catalog | Types without a release |
| Delivery order, order line | Process and part (ordering archetype) | Only the participation vocabulary is modeled here |

## Unmapped Concepts
None identified.

## Clarifying Questions & Answers
- Role forms → classification + participation (R)
- Representation → governed catalog (R: "without a release")
- Same name in two forms → none; "customer" is classification only (I)
- Validity on assignments → none (assumed)
- Eligibility enforcement → refuse at order confirmation; advisory at assignment of "courier partner" (A)
- Participant identity → snapshot for the ward contact; reference for organizations (A)
- Supersession → additive (assumed)
- Cardinalities → orderer exactly one, payer at most one, executor exactly one, receiver any; per line receiver and ward contact any (A)
- Unregistered participants → ward contact only (R)
- Re-check on expiry → flag by nightly reconciliation, do not auto-withdraw (A)

## Subjects & Role Forms

| Subject kind | May hold roles? | Notes |
|--------------|-----------------|-------|
| Organization (hospital, clinic, group, courier company) | Yes | |
| Organizational unit (ward) | Yes — receiver only | |
| Person (ward contact) | Yes — ward contact only; may be unregistered | Snapshot |

| Role | Form | Answers | Same name elsewhere? |
|------|------|---------|----------------------|
| customer | classification | "is S a customer?" | No |
| courier partner | classification | "which couriers may we route to?" | No |
| orderer | participation (delivery order) | "who placed this order?" | No |
| payer | participation (delivery order) | "whom do we invoice?" | No |
| executor | participation (delivery order) | "who is contractually responsible?" | No — but see Applicability (requires the courier partner classification) |
| receiver | participation (order, overridable per line) | "where does this line go?" | No |
| ward contact | participation (line only) | "whom do we call on site?" | No |

## Representation & Catalog
Strategy: governed catalog — partner categories change quarterly without a release.

| Role type | Description | Constraints | Extensions | Requirements |
|-----------|-------------|-------------|------------|--------------|
| customer | Buys deliveries or packaging | organizations only | none | none |
| courier partner | May be routed deliveries | organizations only | none | medical-device transport capability (see Eligibility) |
| orderer / payer / executor / receiver / ward contact | Participation roles of a delivery order | see Applicability | none | executor: as courier partner |

## Applicability & Role-Set Policies

| Role | May be held by | Requires / forbids | Checked at | Removal policed? |
|------|----------------|--------------------|------------|------------------|
| customer | organizations | — | assignment (I) | no (assumed) |
| courier partner | organizations | — | assignment; eligibility advisory (A) | no (assumed) |
| orderer, payer | organizations | — | order build (I) | frozen with the order |
| executor | organizations | requires courier partner (I) | order build; eligibility at confirmation (A) | frozen |
| receiver | organizations, units | — | order/line build (I) | frozen |
| ward contact | persons | — | line build (I) | frozen |

| Scope | Role | Shape | Source |
|-------|------|-------|--------|
| order | orderer | exactly one | (A) |
| order | payer | at most one — absent means the orderer pays | (A); the default is (X — confirm) |
| order | executor | exactly one | (A) |
| order | receiver | any | (A) |
| order | ward contact | forbidden | (I) |
| line | receiver | any | (A) |
| line | ward contact | any | (A) |
| line | orderer, payer, executor | forbidden | (I) |

## Scopes & Participation
Process: delivery order. Parts: order lines.
Participant identity: reference for organizations and wards; snapshot for the ward contact (name, phone) — never updated after the order is placed. Unregistered participants: ward contact only; no order history for contacts.
Effective-roles query: on the order; inherit every role not named on the line, override by role, add line-only roles.

| Role | Scope | Source of assignment | Source tag (R/A/I/X — Step 9.5) |
|------|-------|----------------------|-----|
| orderer | order | the command's security context | (R) |
| payer | order | chosen by the orderer; defaults to the orderer | (A) / (X — confirm default) |
| executor | order | routing by destination region among eligible courier partners | (R) |
| receiver | order + line | orderer by default; line override per ward | (R) |
| ward contact | line | entered on the line | (R) |

Walkthrough — order 1041: St. Anne Hospital = orderer + receiver; Anne Group = payer; MedExpress = executor.
Line 1 (gloves): no line roles → inherits all four.
Line 2 (ultrasound unit): receiver = Cardiology Ward; ward contact = Dr K. (snapshot).
Effective roles for line 2: orderer St. Anne, payer Anne Group, executor MedExpress, receiver Cardiology Ward, ward contact Dr K.

## Eligibility
Enforcement: executor checked at order confirmation (refuse); courier partner checked as an advisory query at assignment and re-checked by a nightly reconciliation that flags partners whose medical-device transport capability has lapsed. Requirements provider: the catalog entry for courier partner, parameterized with the destination region.

| Role type | Capability required | Scope requirements | Match rule (any held capability × all scope requirements) | What-is-missing exposed? |
|-----------|---------------------|--------------------|-----------|--------------------------|
| courier partner / executor | medical-device transport (currently valid) | certifications ⊇ {ISO 13485}; handling ⊇ {cold chain}; regions ⊇ {destination region} | any one held capability meeting all three | yes — shown to the router and the partner manager |

Worked check, destination Mazovia:
- MedExpress — certifications {ISO 13485, ISO 9001}, handling {cold chain}, regions {Mazovia, Silesia}, capability valid → eligible
- QuickVan — certifications {ISO 9001}, handling {ambient}, regions {Mazovia}, capability valid → not eligible; missing: certification ISO 13485; handling cold chain; regions satisfied
- ColdLine — certifications {ISO 13485}, handling {cold chain}, regions {Silesia}, capability valid → not eligible; missing: region Mazovia
- NorthMed — certifications {ISO 13485}, handling {cold chain}, regions {Mazovia}, but the capability's validity ended last month → not eligible; an invalid capability is invisible to the check, so every clause fails at once

## Lifecycle, Time & Boundaries
Events: role assigned / withdrawn / skipped(duplicate, missing). Supersession: additive. Placement: classification roles inside the organization's identity aggregate; participation roles inside the delivery order; snapshots inside the order.
Validity on assignments: none; history through events. Changes after confirmation: executor may be replaced by a new confirmation only (assumed).
Consumers: routing asks "eligible courier partners for region"; invoicing asks "payer of order"; the partner portal asks "does S hold courier partner?". No permissions modeled.

## Implementation Notes
- (X — confirm) payer defaults to the orderer when absent
- (assumed) no validity on assignments; no removal policy on classification roles
- Deliberately not modeled: relationship roles; packaging orders as a separate process (their participation roles are assumed identical to delivery orders)
- Capabilities (certifications, handling, regions, validity) belong to the party archetype; this model reads them through the requirements provider
- Ownership: identity of hospitals, wards and couriers → party model; the delivery order and its lines → ordering model; every role fact above → this model
```

# archetype-modeling

Skills for taking a set of domain requirements and finding the known shapes inside it.

An **archetype** is a recurring model that many domains share once you strip away their
vocabulary: anything that accumulates and is consumed is an accounting model; anything
whose value is computed from rules is a pricing (valuation) model; anything planned and
then measured against what happened is a plan-vs-execution model; anything whose outcome
depends on the arrangement of connections rather than a list of steps is a graphs model; any number that
means nothing without its unit of measure is a quantity model; anything limited that requests
compete for, hold and release is an inventory model; anything the same subject counts as in
different situations is a roles model. Each mapper skill runs
a **fit test** first, and a "this archetype does not apply" answer is a valid, useful one.

## Skills

| Skill | Use it when |
|-------|-------------|
| `accounting-archetype-mapper` | A resource accumulates or is consumed — money, points, quota, inventory, time — and you need auditability, reversibility, and history |
| `pricing-archetype-mapper` | A value is *computed* from rules and context, split among stakeholders, and past results must stay reproducible |
| `party-archetype-mapper` | People and organizations play several roles, relate to each other, hold identifiers, addresses and capabilities |
| `roles-archetype-mapper` | The same subject counts as different things — on its own, toward another subject, or inside a process — with role forms, representation strategy, applicability and cardinality policies, scopes, capability-based eligibility and the boundary with permissions |
| `ordering-archetype-mapper` | Someone submits a request that is confirmed, priced, allocated and fulfilled by others — an order as a record of intent orchestrating the other archetypes |
| `product-archetype-mapper` | Something is offered, performed or settled according to a definition with instances, variants, tracking, catalog entries, relationships, applicability rules and packages |
| `plan-vs-execution-archetype-mapper` | Something was supposed to happen and something did — a schedule, target, budget or promise compared with reported facts under a tolerance, with re-planning, what-if simulation and plan history |
| `graphs-archetype-mapper` | Relations between things decide what happens — swaps that only work as a group, journeys with alternative routes, influence that spreads to uninvolved parties, ordering and parallelism, a single node holding everything together |
| `quantity-archetype-mapper` | Amounts carry units — kilograms, pieces, minutes, gigabytes, money — with precision, rounding, allowed operations, conversions, rates and percentages that must not go wrong |
| `inventory-archetype-mapper` | Requests compete for something limited — specimens, pool amounts, time slots, equipment — that is claimed by an owner, held until released or expired, reserved, queued for when there is not enough, and consumed |
| `archetype-scanner` | You don't yet know which archetypes apply — runs every mapper in parallel and merges the results into one report |
| `archetype-skill-builder` | You have a reference implementation or training transcript for a new archetype and want a mapper skill extracted from it |
| `context-distiller` | You want to know where domain concepts can safely collapse into one abstraction, and where a shared name hides two different things |

All thirteen produce **modeling artifacts** — models, maps, concept mappings — not
implementation code.

## Typical order

```
context-distiller     →  what are the real concepts and boundaries?
archetype-scanner     →  which archetypes fit these requirements?
<name>-archetype-mapper  →  full model for each archetype that fit
```

`archetype-skill-builder` is the meta-skill: it grows the set of mappers the scanner
knows about. Registering a new archetype means adding a row to the registry table in
`archetype-scanner/SKILL.md`.

## Note on invocation

Installed as a plugin, these skills are plugin-qualified:
`archetype-modeling:archetype-scanner`.

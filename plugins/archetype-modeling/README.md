# archetype-modeling

Skills for taking a set of domain requirements and finding the known shapes inside it.

An **archetype** is a recurring model that many domains share once you strip away their
vocabulary: anything that accumulates and is consumed is an accounting model; anything
whose value is computed from rules is a pricing (valuation) model. Each mapper skill runs
a **fit test** first, and a "this archetype does not apply" answer is a valid, useful one.

## Skills

| Skill | Use it when |
|-------|-------------|
| `accounting-archetype-mapper` | A resource accumulates or is consumed — money, points, quota, inventory, time — and you need auditability, reversibility, and history |
| `pricing-archetype-mapper` | A value is *computed* from rules and context, split among stakeholders, and past results must stay reproducible |
| `archetype-scanner` | You don't yet know which archetypes apply — runs every mapper in parallel and merges the results into one report |
| `archetype-skill-builder` | You have a reference implementation or training transcript for a new archetype and want a mapper skill extracted from it |
| `context-distiller` | You want to know where domain concepts can safely collapse into one abstraction, and where a shared name hides two different things |

All five produce **modeling artifacts** — models, maps, concept mappings — not
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

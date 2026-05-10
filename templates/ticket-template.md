# Ticket template

A ticket is a self-contained unit of work an agent can complete in one run (~30-60 min).
The size sweet spot is **10-25 files / one logical feature slice / no more than ~5 cross-file invariants**.

## Header

```
# Ticket: <module> — <NN><sub> <Short Title>

## Summary

One paragraph. What lands when this ships, what's deferred, what depends on it.
Example: "Implement the auth module's hard-constraints aggregate (root + 4 children + audit log)
plus GET / PUT / audit-log REST endpoints. Per `lld/preference.md`. Does NOT include
HardConstraintFilterService (deferred to 01b) or @ValidDietaryIdentity custom validator
(deferred to 01c)."
```

## Behavioural spec

Numbered list of invariants. Each is a single thing the implementation must guarantee.
Aim for ≥10 invariants on a non-trivial ticket. Each invariant should be testable.

- Don't restate the LLD verbatim — link to it.
- DO call out anywhere the ticket overrides the LLD (and why).
- Group by sub-flow (registration, login, audit, validation) — not by file.

## OpenAPI spec excerpt (if HTTP-facing)

The exact YAML chunks to add. The agent will splice these into `openapi.yaml` literally.

**Spring Page<T> schema shape** — use the flat form, not a nested `page: { number, size }` object. Spring Boot 3.2.5+ serialises `Page<T>` to flat properties; nesting them silently doesn't match. Always include `additionalProperties: true` so swagger-parser tolerates Spring's `pageable` / `sort` extras:

```yaml
FooDtoPage:
  type: object
  additionalProperties: true
  required: [content, totalElements, totalPages, number, size]
  properties:
    content:
      type: array
      items: { $ref: '#/FooDto' }
    totalElements: { type: integer, format: int64 }
    totalPages: { type: integer }
    number: { type: integer }
    size: { type: integer }
    first: { type: boolean }
    last: { type: boolean }
    empty: { type: boolean }
    numberOfElements: { type: integer }
```

For nullable scalar/object/enum properties: never use `$ref + nullable: true` (sibling keywords on `$ref` are silently ignored by swagger-parser). Inline the type:

```yaml
status:
  type: string
  enum: [STOCKED, LOW, OUT]
  nullable: true

freezerExtension:
  type: object
  nullable: true
  properties: { ... inline the whole object schema ... }
```

## Edge-case checklist

A bulleted list. Each box maps 1:1 to a test the agent must write.

- [ ] GET on non-existent aggregate → 404 with module-specific ProblemDetail type
- [ ] PUT with stale `expectedVersion` → 409 (concurrent-update ProblemDetail)
- [ ] PUT replacing all child collections cleanly via cascade + orphanRemoval
- [ ] Anonymous request → 401 (filter chain rejects before controller)
- [ ] Authenticated request for user A cannot read user B's aggregate
- ...

This is the contract for what "done" means. The agent's report should explicitly tick each.

## Files this ticket touches

A literal list of file paths under `src/`, with `new` / `modified` / `deleted` annotations.
Crucial for two reasons:

1. The agent uses this as a checklist, not a hint.
2. The reviewer can sanity-check scope at a glance.

```
src/main/resources/db/migration/V<timestamp>__<module>_<feature>.sql                new
src/main/java/com/example/<project>/<module>/...                                    new
src/main/java/com/example/<project>/<module>/api/controller/...                     new
src/test/java/com/example/<project>/<module>/<Feature>FlowIT.java                   new
...
```

## Dependencies

Hard dependencies (tickets that must be merged first) and soft dependencies (siblings
that may run in parallel but share files).

```
- **Hard dependency**: <previous-ticket-id> — uses <specific-thing> from it.
- **Sibling tickets running in parallel**: <list> — see scope walls.
```

## Acceptance / DoD

- [ ] `./mvnw -Dspotless.check.skip=true verify` passes locally on agent's worktree
- [ ] CI green on the PR (build + spotless + OpenAPI lint + pitest if labelled)
- [ ] Mutation score ≥70% on the security-critical / domain-critical packages (if specified)
- [ ] All edge-case checklist items ticked
- [ ] No regressions on existing tests

Squash-merge with: `<conventional-commit-style commit message>`

## What's NOT in scope

Explicit. Saves the reviewer asking "why didn't you do X?".

---

## Sizing — when to split

If the ticket has any of these, consider splitting:

- More than 25 files
- More than 15 invariants in the behavioural spec
- More than one cross-cutting concern (e.g., auth + throttling + password change)
- Estimated agent runtime > 1 hour

The source-project's auth-01 was 27 invariants / ~40 files / 2-day estimate. Split into
01a/01b/01c (each ~12 files / ~10 invariants / 30-45 min). Velocity tripled.

## Sizing — when NOT to split

Don't split for the sake of splitting:

- A coherent aggregate (e.g., HardConstraints + 4 children + audit log) belongs in one
  ticket. Splitting child entities into separate tickets just creates merge churn.
- A migration + entity + repo + service + endpoint for one new feature is one ticket.
- Test-only refactors that touch many files but have one logical purpose stay together.

The test: **can the agent write this in one focused pass without losing the thread?** If
yes, one ticket. If no, split.

# Decision 0009 — Wave 2 round 6 (rate-limit recovery, cross-module SPI second wiring, parent-fix asymmetry)

**Date**: 2026-05-11 (late evening)
**Status**: Round 6 complete — 4 implementation tickets shipped
**Source**: TwoLaugh/foodSystem PRs #31–#34

## What landed

| Ticket | PR | Files | CI rounds | Parent fixes |
|---|---|---|---|---|
| household-01f (slot-configuration planner view) | [#31](https://github.com/TwoLaugh/foodSystem/pull/31) | 11 | 1 | 0 (clean ship) |
| nutrition-01f (NutritionCalculationService + RecipeNutritionWriter SPI) | [#32](https://github.com/TwoLaugh/foodSystem/pull/32) | 22 | 2 | 1 (@MockBean collateral on 3-interface RecipeServiceImpl) |
| provisions-01f (ProvisionForPlannerService.getBundle) | [#33](https://github.com/TwoLaugh/foodSystem/pull/33) | 11 | 3 | 2 (table name `_log` mismatch, OpenAPI nullable+$ref regression, query-count threshold) |
| recipe-01f (RecipeWriteApi SPI + adaptation events) | [#34](https://github.com/TwoLaugh/foodSystem/pull/34) | 14 | 1 | 0 (clean ship; RecipeNutritionWriterImpl deferred to follow-up commit per parallel-safety) |

**Round-6 wall-clock**: ~3.5 hours including a hard rate-limit pause in the middle that killed the first batch of agents before any wrote code. Re-spawned all 4 after reset.

## The rate-limit recovery worked

Halfway through spawning the 4 agents, the daily usage limit kicked in. All 4 agents returned within seconds with "You've hit your limit · resets 10:20pm". Each worktree was untouched — agents hadn't even read the ticket yet.

After the reset, re-spawning the same 4 agents with the same prompts worked cleanly. **No code was lost**; the worktrees were still in place, the tickets were still committed, just had to re-launch.

**Lesson**: rate limits are a recoverable interruption like session restart. Pattern: re-spawn with same prompts on the same worktrees. Document in playbook.

## The new gotcha: @MockBean collateral damage on multi-interface impls

`RecipeServiceImpl` implements `RecipeQueryService + RecipeUpdateService + RecipeSubstitutionRecorder` (one class, three facades). The nutrition-01f IT did `@MockBean private RecipeQueryService recipeQueryService` to stub just the query side. But Mockito's `@MockBean` REMOVES the entire real bean (the single `RecipeServiceImpl`) and substitutes a mock that ONLY satisfies the explicitly-named interface. Result: Spring can no longer find `RecipeUpdateService` or `RecipeSubstitutionRecorder` beans, `RecipeModule` fails to wire, application context refuses to start, ALL 4 tests in the IT error out with "expected at least 1 bean which qualifies as autowire candidate".

**Fix**: when `@MockBean` a multi-interface impl, also `@MockBean` every other interface it provides. Add stubs explicitly so Spring has beans to wire elsewhere.

```java
// Before — context-load failure
@MockBean private RecipeQueryService recipeQueryService;

// After — three explicit stubs (one per interface RecipeServiceImpl implements)
@MockBean private RecipeQueryService recipeQueryService;
@MockBean private RecipeUpdateService recipeUpdateService;
@MockBean private RecipeSubstitutionRecorder recipeSubstitutionRecorder;
```

This is a class of bug that didn't show up in earlier rounds because no test had previously `@MockBean`'d an interface where the same `@Service` class implemented multiple SPI surfaces. Worth baking into the agent-prompt-template.

## Recurring OpenAPI 3.0 nullable+$ref regression

Provisions-01f's planner-bundle DTO had `spendTracking: { $ref: '#/BudgetSpendTrackingDto', nullable: true }`. Same exact round-1 trap. Round-1 gotcha list explicitly says "nullable: true next to $ref is silently ignored — inline the type".

The ticket-writer agent flagged this convention; the implementation agent applied the inline pattern in MOST schema fields but missed this one (because the standalone `BudgetSpendTrackingDto` was already defined as a top-level schema and reusing it felt natural). Fix: inline the schema definition directly in the parent DTO's `properties:` block rather than referencing the standalone type when nullable.

**Cumulative gotcha-incidence count for OpenAPI 3.0 nullable+$ref**: Round 1 (3 occurrences fixed parent-side), Round 4 (1), Round 6 (1). Still leaks despite being in the agent prompt template. The pattern is sticky because reusing a named schema feels like good DRY hygiene — the agent has to actively choose to copy-paste-inline. Worth strengthening the prompt language.

## Cross-module SPI v2: parallel-safety via deferred impl

Round 6 introduced the second cross-module SPI: nutrition defines `RecipeNutritionWriter`, recipe wires the impl. Same pattern as round 5's household-defines / recipe-wires-impl. But this round's twist: **the impl class won't compile until the SPI interface is on classpath**. With both modules in parallel branches, the SPI interface is in branch A, the impl is in branch B; B can't import A's interface until A merges.

The prompts explicitly told recipe-01f to SKIP shipping the impl ("defer to a follow-up commit after nutrition-01f merges"). Recipe agent honored this — shipped RecipeWriteApi + events but no RecipeNutritionWriterImpl. After both PRs land, a small bridge commit can wire the impl.

**Lesson**: cross-module SPI is now a proven pattern at 2-tier depth (household→nutrition in round 5, nutrition↔recipe in round 6). The parallel-friendly recipe: define interface in consumer module, ship Noop fallback for parallel landing, wire real impl in a follow-up commit.

## Parent-fix asymmetry

| Module | Round-6 parent fixes | Cumulative pattern |
|---|---|---|
| household | 0 | Has been the "easiest" module across all rounds |
| recipe | 0 | Clean shipping when no cross-module write back |
| nutrition | 1 | Multi-pattern tickets (calc + SPI + listener) — consistently the heaviest |
| provisions | 2 | Surprise — usually clean; round 6 hit OpenAPI nullable regression + threshold |

Total parent-fix incidents across rounds 1-6: ~16. The gotcha list keeps growing but the per-round incident count has stayed in the 2-4 range; we're not seeing a steady decline because every round introduces a new pattern class (round 5 was SPI; round 6 was multi-interface @MockBean).

## What's next

Per the round-1+...+6 deferral maps, remaining Wave 2 sub-tickets:

- household: 01g (none enumerated — possibly done?)
- nutrition: 01g floor-gate, 01h aggregation
- provisions: 01g cook events, 01h grocery import, 01i staples, 01j batch-cook, 01k expiry, 01l feedback, 01m pantry gate
- recipe: 01g promotion/archive, 01h embeddings, 01i search, 01j helpers, 01k AI tag inference

~12 sub-tickets in Wave 2. At the round-6 pace (~80 min per round of 4), that's ~3-4 more rounds to finish backend Wave 2. Then Wave 3 (orchestrators: planner, discovery, feedback).

Also pending: a small bridge commit to wire `RecipeNutritionWriterImpl` now that both nutrition-01f and recipe-01f are merged on main.

## Implications for ai-workflow templates

1. **`templates/agent-prompt-template.md`** — add @MockBean multi-interface gotcha: when stubbing an interface, check if the real `@Service` class implements multiple SPI interfaces; if yes, add `@MockBean` for each.
2. **`templates/agent-prompt-template.md`** — strengthen the OpenAPI 3.0 nullable+$ref language to say "if a field is nullable and references a named schema, INLINE the schema in the parent's properties block instead of using `$ref + nullable: true`. Even if you defined the standalone schema for documentation, don't reuse it via `$ref` when the field is nullable."
3. **`playbook/parallel-agent-coordination.md`** — add rate-limit recovery section: same pattern as session restart. Worktrees are durable; re-spawn after reset.

# Decision 0010 — Wave 2 round 7 (`@TransactionalEventListener` propagation rule; SPI bridge end-to-end)

**Date**: 2026-05-12
**Status**: Round 7 complete — 3 modules shipped (household has no 01g; 3-way parallel)
**Source**: TwoLaugh/foodSystem PRs #35–#37

## What landed

| Ticket | PR | Files | CI rounds | Parent fixes |
|---|---|---|---|---|
| nutrition-01g (NutritionFloorGateService) | [#35](https://github.com/TwoLaugh/foodSystem/pull/35) | 13 | 1 | 0 (clean ship) |
| provisions-01g (cook events + sealed `ProvisionChangedEvent` + deduction engine) | [#37](https://github.com/TwoLaugh/foodSystem/pull/37) | 47 | 1 | 0 (clean ship — the biggest ticket of round 7) |
| recipe-01g (promotion + archive + SPI bridge from round 6) | [#36](https://github.com/TwoLaugh/foodSystem/pull/36) | 23 | 3 | 2 (AFTER_COMMIT tx propagation) |

**Round-7 wall-clock**: ~1h20min from spawn to all-merged.

## The new gotcha: `@TransactionalEventListener` propagation rule

The recipe-01g SPI bridge IT (`RecipeNutritionWriterImplIT.endToEnd_manualEdit_triggersNutritionListener_andBridgeWritesNutritionStatus`) failed because:

1. Manual edit on a recipe publishes `RecipeUpdatedEvent`.
2. Nutrition's `RecipeEventListener.onRecipeUpdated` runs `@TransactionalEventListener(phase = AFTER_COMMIT)`.
3. AFTER_COMMIT means the publisher's tx has committed and closed; there is no active transaction.
4. The listener body does JPA reads (loading the version) and eventually calls `RecipeNutritionWriter.writeNutritionPerServing` which calls `RecipeWriteApi.updateNutritionStatus` — that method is `@Transactional`, so Spring opens a new tx for IT.
5. **But**: in between (step 4 onwards), the listener body itself touches JPA-managed entities outside a tx, which throws `TransactionRequiredException`.

**First attempt** (parent fix #1): add plain `@Transactional` (default `REQUIRED` propagation) to `onRecipeUpdated`. Compiles, but at Spring context-load:

```
@TransactionalEventListener method must not be annotated with @Transactional
unless when declared as REQUIRES_NEW or NOT_SUPPORTED
```

ALL ITs across ALL modules failed to load context — every IT errored "Failed to load ApplicationContext". This is a fail-fast at startup; the rule is enforced strictly.

**Second attempt** (parent fix #2): `@Transactional(propagation = Propagation.REQUIRES_NEW)`. Green.

**Round-5 retro flagged a related rule** ("`@TransactionalEventListener` cannot be paired with `@Transactional(SUPPORTS)`") — turns out the ACTUAL rule is broader: only `REQUIRES_NEW` and `NOT_SUPPORTED` are allowed when both annotations are on the same method. The retro had a partial truth; the full rule needs to go in the prompt template.

## What worked

- **provisions-01g shipped clean despite being the biggest ticket of round 7** (~47 files, sealed event refactor + FIFO deduction engine + 3 consumption flows + dedupe table). The ticket's pre-approved escape-hatch split wasn't needed; the verify loop converged in 2 agent-side iterations. The agent navigated the sealed-base refactor (adding existing concrete events to permits) without dropping any.
- **The SPI bridge from round 6 finally completes end-to-end**. Recipe → nutrition listener → `RecipeNutritionWriter` SPI → `RecipeWriteApi.updateNutritionStatus` flow proven in IT (after the propagation fix). The "ship in two waves" pattern (interface in module A, impl in module B follow-up) is fully validated.
- **3-way parallel** (vs 4-way) was clean. No household 01g needed; running 3 instead of 4 just means slightly less Docker contention and slightly faster local turnaround.

## What's left in Wave 2

Per the cumulative deferral maps:

- household: **DONE** for Wave 2 ✓
- nutrition: 01h aggregation
- provisions: 01h grocery import, 01i staples, 01j batch-cook, 01k expiry, 01l feedback, 01m pantry gate
- recipe: 01h embeddings, 01i search, 01j helpers, 01k AI tag inference

~10 sub-tickets remaining. At the round-7 pace (~80 min per round of 3), that's ~3-4 more rounds to finish Wave 2 backend.

Note: nutrition has only ONE remaining sub-ticket (01h aggregation) and household is done. Future rounds will run 3-way at most, possibly dropping to 2-way as more modules finish.

## Implications for ai-workflow templates

1. **`templates/agent-prompt-template.md`** — replace the round-5 partial-truth gotcha ("`@TransactionalEventListener` cannot be paired with `@Transactional(SUPPORTS)`") with the full rule:

   ```
   @TransactionalEventListener method must use Propagation.REQUIRES_NEW or
   Propagation.NOT_SUPPORTED if also annotated @Transactional. The default
   (REQUIRED) and SUPPORTS are rejected at context-load — failure is fail-fast,
   blocks every IT across the project. For AFTER_COMMIT listeners that need a
   tx (any JPA read/write inside the body), use:

       @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
       @Transactional(propagation = Propagation.REQUIRES_NEW)
       public void onSomeEvent(SomeEvent event) { ... }
   ```

2. **`templates/agent-prompt-template.md`** — add a positive guidance line for "AFTER_COMMIT listeners that do JPA": if your listener calls into a Spring-managed bean's `@Transactional` method, that target will open its own tx — but ANY JPA read in the listener body BEFORE that call is also outside a tx and will fail. Add `@Transactional(REQUIRES_NEW)` on the listener method itself.

3. **`playbook/parallel-agent-coordination.md`** — note that with one module retiring per round (household this round), parallelism naturally narrows. 3-way is fine; 2-way is still worthwhile for the time saved on Docker-contention-induced flakes.

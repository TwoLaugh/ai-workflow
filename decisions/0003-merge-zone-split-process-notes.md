# Decision 0003 — Merge-zone split (process timings + new bottlenecks)

**Date**: 2026-05-09
**Status**: Validated (PR open at TwoLaugh/foodSystem#10; awaiting CI)
**Source**: TwoLaugh/foodSystem `refactor/01-split-merge-zones`

## Why this doc

`decisions/0002-speedup-validation-results.md` measured single-ticket speedup. This doc tracks the meta-cost of the parallel-prep refactor — the work we're doing once so that 5+ later parallel rounds avoid merge-zone churn — and any bottlenecks that surface during it.

## What's being refactored

Three "merge zones" in the source project being split into per-module files:

1. `openapi.yaml` (703 lines, single file) → entry + `paths/{module}.yaml` + `schemas/{module}.yaml` via `$ref`, validator switched from inline-string loading to URL loading.
2. `GlobalExceptionHandler.java` (331 lines, all exception types) → cross-cutting only + per-module `<Module>ExceptionHandler.java` classes + shared `ProblemDetailSupport` helper.
3. `archunit/ModuleBoundaryTest.java` (per-module repo rules in one file) → per-module `<module>/<Module>BoundaryTest.java` files; cross-cutting rules stay in shared file.

## Timings (parent-side)

| Phase | Wall time | Note |
|---|---|---|
| Inspect existing merge-zone files | ~3 min | Read of 4 files (GlobalExceptionHandler, openapi.yaml structure, ArchUnit, validator config). Could've been 1 min if I'd known the line ranges upfront, but the discovery was load-bearing for verbatim snippets. |
| Write the refactor ticket with verbatim shape snippets | ~6 min | ~700-line ticket; includes 3 full target Java files, 1 entry-point YAML shape, 2 per-module YAML examples. |
| Spawn agent | (instant) | But hit a wall — see "New bottleneck" below. |
| Agent self-verify + execute | ~17.6 min | 3 verify iterations. Caught a real `@Order` bug (the ticket said advices wouldn't conflict — they did, because `@ExceptionHandler(Exception.class)` is itself a catch-all that swallows module-specific exceptions when ordering is left default). |
| Parent verify + diff review | ~4 min | mvn verify 3:14, spotless:check 6s, diff review ~30s. No issues — agent's report matched the diff exactly. |
| Commit + push + PR open | ~30s | gh CLI; CI started immediately. |
| **Total parent-side authoring + orchestration** | **~31 min** | Of which ~9 min was active typing (ticket writing + commit msg) and ~22 min was wait/review of agent and verify output. |

## New bottleneck found during this round

**Worktree isolation isn't always available.** The Agent tool's `isolation: "worktree"` flag failed with:

> Cannot create agent worktree: not in a git repository and no WorktreeCreate hooks are configured.

…even though the project IS a git repo on a feature-branchable state. The harness's "is a git repository" detection happens at session start and isn't reactive to later state. In our case the auto-detected env had `Is a git repository: false` despite `.git/` being present in the working directory.

**Fix used**: ran the refactor on a feature branch (`refactor/01-split-merge-zones`) directly — no worktree, no isolation. Acceptable for refactors where rollback is `git checkout main && git branch -D <branch>` if anything goes wrong.

**Implication for `playbook/parallel-agent-coordination.md`**: the doc currently assumes worktree isolation as default. Add a fallback section: "If the harness can't create a worktree, fall back to per-branch isolation. For N parallel siblings, this means N separate sessions or running them serially with branch-switches in between — losing the parallelism win."

This is a real cap on parallelism in the current environment. If running 4 parallel agents in Wave 2 turns out to also fail worktree creation, the practical answer is: run them in 2 sessions of 2 (not 4 of 1) and accept ~2× wall-clock instead of ~4×. Or fix the harness's git detection.

## Other bottlenecks observed

- **Verbatim shape snippets are still expensive to author**. ~6 min to write the ticket, vs ~30s for a "see file X for shape" pointer. Pays off at agent runtime if it saves multiple file-read tool calls per snippet, but the parent-side authoring cost should not be ignored. Net positive when the ticket is one of many parallel agents will follow; net negative for one-off tickets.
- **Three orthogonal sub-tasks in one ticket**: the YAML split, the exception-handler split, and the ArchUnit split don't depend on each other. We could have run them as 3 sub-agents in parallel — but each is 5-10 min of agent work, so the orchestration overhead probably exceeds the savings on a refactor of this size. Worth revisiting if a future refactor has 5+ orthogonal pieces.

## What the refactor actually unlocks

Per-Wave-2-round savings (rough projection):

- **No openapi.yaml conflicts**: each parallel agent adds new `paths/<module>.yaml` + `schemas/<module>.yaml` plus exactly two lines in entry `openapi.yaml` (one path ref, one schema ref). Worst case: 4 agents × 2 line additions = 8 trivial conflicts in the entry file. Resolution: alphabetical sort, ~30 seconds. Down from ~3-5 min per round of merging the prior monolith.
- **No `GlobalExceptionHandler` conflicts**: each module either creates its own `<Module>ExceptionHandler.java` (no conflict) or doesn't throw new exceptions (no edit at all). Down from ~2-3 min per round.
- **No `ModuleBoundaryTest` conflicts**: each module adds its own `<Module>BoundaryTest.java`. Down from ~1 min per round.

Net: ~6-9 min saved per parallel-merge round. With 5-7 Wave 2 rounds expected, the refactor pays for itself after round 4-5.

## Implications for ai-workflow templates (validated, can be patched now)

1. **`templates/agent-prompt-template.md` — `@Order` rule for `@RestControllerAdvice`**: Add to the "shape snippets" guidance: when a project uses multiple `@RestControllerAdvice` beans AND any of them has an `@ExceptionHandler(Exception.class)` catch-all, the spec MUST require `@Order` annotations:
   - The catch-all advice → `@Order(Ordered.LOWEST_PRECEDENCE)`
   - Module-specific advices → `@Order(Ordered.HIGHEST_PRECEDENCE)`

   The Spring docs are vague on this and "default order" varies subtly between Spring versions. Specifying explicitly removes the trap. This was the only iteration the verify loop caught — exactly the class of bug the loop is for, but worth eliminating in the prompt for next time.

2. **`playbook/parallel-agent-coordination.md` — worktree fallback**: add a short section: "If the harness reports `not in a git repository` despite `.git/` being present, the worktree-isolation feature is unavailable. Fall back to per-branch isolation: the parent creates a feature branch, the agent works in-place on it, the parent merges. Cost: agents can't run in true parallel — they share the working tree. For Wave-2-style parallel batches, this means running siblings serially and amortising the parent-side review."

3. **`playbook/verify-loop.md` — "iteration is normal" calibration**: the loop hit 3 iterations on a refactor where I had high confidence the prompt was complete. Two were the `@Order` ordering issue (one fix attempt didn't go far enough; the second resolved it). This is consistent with the playbook's "1-3 typical" pacing — confirming the cap of 5 is generous, not stingy.

## What was correctly NOT in the ticket (validated as safe to drop)

- "Read the LLD docs first." Agent didn't need to. The verbatim snippets carried all the structural intent.
- "Re-discover the existing file layouts." Agent confirmed it Read'd the existing files exactly as needed (to know what to delete from `GlobalExceptionHandler.java` and what to keep).
- "Run unit tests before integration tests." Conventional `mvn verify` order works fine.

## Open questions for next round

- Does this refactor's "add new files only" property hold up in practice when 4 parallel agents each add their own module entry + schemas, or does the entry-file `paths:` mapping become a merge zone again? Test in Wave 2 round 1 (household + nutrition + provisions + recipe).
- Is `@Order(HIGHEST_PRECEDENCE)` correct for ALL future module advices, or do we need a tier system (module-A vs module-B priority for cross-module exceptions)? Probably no — module exceptions don't overlap, so any HIGHEST_PRECEDENCE ordering should be fine.

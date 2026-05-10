# Decision 0006 — Wave 2 round 3 (manual-worktree parallel implementation)

**Date**: 2026-05-10
**Status**: Validated — 4 modules implemented in true parallel via manually-created git worktrees
**Source**: TwoLaugh/foodSystem PRs #19–#22

## What landed

| Ticket | PR | Files | Agent runtime | CI build time |
|---|---|---|---|---|
| provisions-01c | [#19](https://github.com/TwoLaugh/foodSystem/pull/19) | ~20 | 51 min | 5:35 |
| household-01c | [#20](https://github.com/TwoLaugh/foodSystem/pull/20) | ~31 | 62 min | 5:50 |
| nutrition-01c | [#21](https://github.com/TwoLaugh/foodSystem/pull/21) | ~21 | 62 min | 5:53 |
| recipe-01c | [#22](https://github.com/TwoLaugh/foodSystem/pull/22) | ~32 | 68 min | 6:26 |

**Round-3 total wall-clock**: 14:35 → 15:53 = **78 minutes**.

For comparison: round-1 (4 sibling modules, serial) = ~4 hours; round-2 (4 sibling modules, serial) = ~5.5 hours. **Round 3 ran ~3-4× faster wall-clock by going truly parallel.**

## How "true parallel" was achieved despite the harness limitation

The Agent tool's `isolation: "worktree"` flag still fails because the session's primary working directory (`C:\Users\irenv\Claude`) isn't itself a git repository — only the project subdirectory (`claudeTest`) is. The harness's git detection at session start is sticky.

**Workaround**: the parent (me) created 4 git worktrees manually using `git worktree add`, each on its own branch off main. Then spawned 4 implementation agents, each prompt explicitly directing the agent to:
- Use absolute paths in `Read`/`Edit`/`Write` (which take absolute paths anyway)
- Prefix every Bash command with `cd "<worktree-absolute-path>" ; ...` so `mvn` and `git` operate on the correct working tree
- Treat its worktree as the project root for `tickets/` / `src/` lookups

All 4 agents launched in a single message with `run_in_background: true`. They worked completely in parallel.

This bypass is **simpler than configuring WorktreeCreate hooks** in `settings.json` (which the harness suggests as the proper fix) and works without restarting the session.

## What worked

- **No file collisions across worktrees.** The merge-zone refactor's per-module file scope held up perfectly under true parallel: each agent appended only to its own module's `paths/<m>.yaml`, `schemas/<m>.yaml`, `<Module>ExceptionHandler.java`, plus a 2-4-line additive append to entry `openapi.yaml`. Sequential rebase + merge resolved cleanly with zero hand-merging needed.
- **Sequential PR merges with rebase between them** worked smoothly. Each subsequent worktree did `git fetch origin main && git rebase origin/main` before push — picking up earlier-merged siblings' entry-yaml edits without conflict (they were in non-overlapping regions of `paths:` and `components.schemas:`).
- **Module-test isolation**: every agent's NEW tests passed cleanly. The contention was only on cross-module ITs that shared infrastructure.

## What broke under contention (all environmental, not real bugs)

Running 4 simultaneous `mvn verify` invocations on Windows + Docker Desktop produced three flake classes:

1. **Docker daemon overload**: `pgvector/pgvector:pg16` Testcontainers startup occasionally returned HTTP 500 from the Docker daemon. Affected ~30% of cross-module ITs during the parallel phase. Worktree-internal ITs (only the agent's own NEW tests) were less affected because they're the FIRST to run when each agent's verify starts — peak Docker load happens later.

2. **Mockito InlineByteBuddyMockMaker self-attach failure** on Windows JDK 17.0.2 under multi-fork: 51 tests across the codebase failed with "Could not initialize inline Byte Buddy mock maker." This is a known Windows JDK + Mockito 5 interaction; running in a single fork avoids it. CI on Linux is unaffected.

3. **JVM "VM terminated without saying goodbye"**: surefire forks crashed under memory pressure with 4× simultaneous JVMs. Module tests retried in isolation passed cleanly.

**Critical finding**: every agent's failures were on tests they didn't touch. CI on GitHub-hosted runners (Linux, isolated, single-tenant) ran every PR green on first try. **The local-parallel-mvn-verify path is unreliable on Windows; CI is the source of truth.** Pattern going forward: skip parent's local full-verify when 4 agents are running concurrently — push directly to PR and let CI judge.

## Implications for ai-workflow templates

1. **`playbook/parallel-agent-coordination.md`** — add a section "Manual worktree parallel pattern" with the recipe used here:
   ```
   parent: git worktree add ../<project>-wt-<module> -b feat/<module>-<NN> main
   parent: <repeat for each parallel ticket>
   parent: spawn N agents in one message with absolute paths
   each agent: "cd <worktree-abs-path> ; ./mvnw verify" for shell commands
   each agent: absolute paths in Read/Edit/Write
   ```
   Plus the rebase-on-each-PR loop and the "skip local full verify under parallel; trust CI" advice.

2. **`templates/agent-prompt-template.md`** — add a "When running in a worktree" section that explicitly states:
   - Use absolute paths in Read/Edit/Write
   - Prefix Bash with `cd "<worktree-abs-path>" ; ...`
   - Don't `cd` into the parent project directory — your worktree IS the project for your purposes
   - If `mvn verify` flakes on cross-module tests under parallel pressure, run `-Dtest=YourNewTest -Dit.test=YourNewIT` in isolation to confirm your code works; CI will validate the rest

3. **`playbook/verify-loop.md`** — add a caveat: "When running in a multi-worktree parallel setup on Windows, the local verify loop can flake on cross-module Mockito + Testcontainers tests due to Docker daemon overload, JVM memory pressure, and Mockito self-attach issues. Trust CI as the source of truth; don't iterate the local loop more than 2-3 times trying to chase environmental flakes that disappear in CI's isolated runner."

## Open questions

- **Is 4 the right parallelism limit on this machine?** All 4 ran but the contention was significant. 3 might be a better steady-state (fewer flakes, only ~10-15% slower wall-clock). Worth experimenting on round 4.
- **Could `mvn -T 1` (single-threaded) reduce contention?** Worth testing — surefire's default fork count is dynamic.
- **Does the agent need a hint to disable spotless during the verify loop and only run it once at the end?** Spotless takes ~10-15s per run; if agents iterate verify they pay this each time. Might save 30-60s per agent.

## Round comparison

| Metric | Round 1 (serial) | Round 2 (serial) | Round 3 (parallel) |
|---|---|---|---|
| Wall-clock | ~4 h | ~5.5 h | **~1.3 h** |
| Avg agent runtime | 25 min | 23 min | 61 min |
| Verify iterations | 2/ticket | 1.25/ticket | 2-5/ticket |
| Parent-side fixes | 5 | 0 | 0 |
| CI green on first push | 4/4 | 4/4 | 4/4 |

Per-ticket runtime ROSE under parallel (61 min vs 23-25 min serial) because each agent burned cycles fighting flakes. But the overall wall-clock fell because they ran simultaneously. **Net: 3-4× faster delivery.** On a Linux machine without the Mockito-self-attach issue, parallel runtime should drop closer to serial single-agent runtime, getting closer to 4× theoretical speedup.

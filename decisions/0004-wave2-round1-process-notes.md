# Decision 0004 — Wave 2 round 1 process notes

**Date**: 2026-05-09
**Status**: Round 1 complete — 4 implementation tickets shipped to main
**Source**: TwoLaugh/foodSystem PRs #11–#14

## What landed

| Ticket | PR | Files | Agent runtime | Verify iter (agent) | Parent fixes | Wall-clock |
|---|---|---|---|---|---|---|
| household-01a | [#11](https://github.com/TwoLaugh/foodSystem/pull/11) | ~28 | 25.5 min | 4 | 0 | ~30 min |
| provisions-01a | [#12](https://github.com/TwoLaugh/foodSystem/pull/12) | ~37 | ~17 min | 0 (denied) | 3 (OpenAPI 3.0 nullable + page schema) | ~35 min |
| nutrition-01a | [#13](https://github.com/TwoLaugh/foodSystem/pull/13) | ~38 | ~35 min | 4 | 0 | ~40 min |
| recipe-01a | [#14](https://github.com/TwoLaugh/foodSystem/pull/14) | ~56 | ~19 min | 0 (denied) | 2 (varchar(32) too narrow for `'user:<uuid>'`) | ~30 min |

**Round-1 total wall-clock**: ~2h35m (15:58 → 23:38, with breaks). Active orchestration time substantially less.

## The headline bottleneck: agent verify access is unreliable

Two of the four agents were denied Bash/PowerShell access by the harness — they never ran `mvn verify`. This forced me into the verify-loop role parent-side. The agents that DID get shell access (household, nutrition) caught real bugs in their own iterations that would otherwise have surfaced parent-side.

**Concrete cost**: provisions-01a took 3 parent iterations (one round-trip per OpenAPI 3.0 schema gotcha). Recipe-01a took 2. That's ~6-10 min of parent-side latency per denied agent, plus context-switching overhead.

**What this means for Wave 2 round 2 onwards**:
- Don't assume the agent's verify gate is load-bearing. Prepare to step in.
- Agents that DO have shell access still benefit from the gotcha-list in the prompt — household and nutrition agents both cited "the gotcha doc said X" when fixing iteration 2.

## OpenAPI 3.0 schema pitfalls — the new gotcha class

Three concrete OpenAPI 3.0 schema bugs surfaced in this round, all on the parent side after the agent shipped its initial pass:

1. **`nullable: true` next to `$ref` is silently ignored.** swagger-parser only honours the ref's target schema; sibling keywords are dropped. Workaround: inline the type for nullable fields rather than ref it.
2. **`allOf` + `nullable: true` parent doesn't work either.** swagger-parser still rejects null when the inner allOf schema doesn't match. Same workaround: inline the schema directly.
3. **Spring `Page<T>` adds `pageable`/`sort` properties** that named page schemas reject as additionalProperties unless explicitly `additionalProperties: true`.

All three are now baked into the gotcha list in `templates/agent-prompt-template.md` (committed alongside this doc).

## LLD spec errors caught at implementation time

Recipe-01a's LLD spec'd `created_by_actor varchar(32)` but the contract value `'user:<uuid>'` is 41 chars. Real spec error. Fix lands in 01a's migration (varchar(64)) — but worth flagging that LLDs aren't infallible. Future ticket-writer agents should compute lengths from format strings, not parrot the LLD's column widths.

## Architectural insight — replaceChildren + unique-index hazard

The nutrition agent caught (in iteration 3 of its self-verify loop) that the `replaceChildren()` then `saveAndFlush` pattern from `PreferenceServiceImpl.updateHardConstraints` does NOT survive when child tables have `(parent_id, business_key)` unique constraints. Hibernate flushes inserts before deletes within one flush; the new rows collide on the unique index before the old ones are gone.

Fix: compute the change-set BEFORE mutating; only mutate + saveAndFlush when there's an actual change. Worth flagging for downstream tickets that add child collections — nutrition-01b/c/e/h, provisions-01b/g, recipe-01b/c/e all have this shape.

## What worked

- **Pre-split of recipe-01a** (deferring `RecipeBranchDto` + `branches[]` field to 01b) was the right call. The agent finished in 19 min (vs the projected 45-60 min); the deferral was clean (`RecipeBranch` entity stays internal, with a JdbcTemplate test asserting its existence).
- **Per-module file scope after the merge-zone refactor** held up across all four parallel-able tickets. Entry `openapi.yaml` had four trivial 2-line additions, no merge conflicts.
- **The household agent's "package-private repos" caveat** was correctly applied by all subsequent agents. The pattern is now established: `public` interfaces enforced by ArchUnit per-module boundary tests.
- **`@Order(HIGHEST_PRECEDENCE)` on module advices** worked correctly across all 4 — no recurrence of the global-catch-all-swallowing-module-exceptions bug from the merge-zone refactor.

## What didn't work as well

- **Worktree isolation never came back.** This continued forcing serial execution. Wave 2 round 1 ran ~4 hours wall-clock for what should've been ~50 min if 4-way parallel had worked.
- **Agent shell-access denial was sporadic** — same harness, same project, two of four agents denied. No pattern I can identify. Adding "if your environment denies Bash/PowerShell, do your best on code, then report explicitly" to the prompt did get the agent to flag clearly rather than silently skip — small win.

## Next-step recommendations for Wave 2 round 2

1. **Round 2's pickable tickets (verify against LLDs first)**: household-01b (settings), nutrition-01b (intake), provisions-01b (equipment + admin lifecycle), recipe-01b (URL import + branches[] DTO). All layer onto round 1's read-by-others contracts.
2. **Don't pre-split unless an LLD module's foundation is genuinely >25 files**. Recipe-01a's pre-split was a clear win because branches were structurally orthogonal. Most other modules' 01b tickets are tighter scope and shouldn't need it.
3. **Update `templates/agent-prompt-template.md` gotchas** with: OpenAPI 3.0 nullable+$ref, OpenAPI 3.0 allOf+nullable+$ref (same trap), Page<> additionalProperties: true, replaceChildren+unique-index hazard, varchar widths must be computed not parroted.
4. **Try worktree isolation one more time** at session start — maybe a fresh session re-detects git state. If it works, round 2 can run truly parallel.

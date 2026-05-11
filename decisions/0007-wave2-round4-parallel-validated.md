# Decision 0007 — Wave 2 round 4 (parallel pattern continues paying back)

**Date**: 2026-05-11
**Status**: Round 4 complete — 4 implementation tickets shipped
**Source**: TwoLaugh/foodSystem PRs #23–#26

## What landed

| Ticket | PR | Files | Agent runtime | CI first-try | Parent fixes needed |
|---|---|---|---|---|---|
| household-01d (member admin) | [#23](https://github.com/TwoLaugh/foodSystem/pull/23) | 19 | ~24 min | red (Spotless) then red (YAML comma) then green | 2: Spotless format + 1 YAML inline-flow comma quoting |
| nutrition-01d (USDA/OFF cache) | [#24](https://github.com/TwoLaugh/foodSystem/pull/24) | 37 | (uncaptured — agent reply timed out during the long gap) | red (2 ITs URL-encoded firewall) then green | 1: IT URL double-encoding fix |
| provisions-01d (supplier products) | [#25](https://github.com/TwoLaugh/foodSystem/pull/25) | 24 | (uncaptured) | green first-try | 0 |
| recipe-01d (branch creation + divergence + revert) | [#26](https://github.com/TwoLaugh/foodSystem/pull/26) | 25 | (uncaptured) | green first-try | 0 |

**Round-4 wall-clock**: ~2 calendar days (with a long overnight gap; effective implementation+CI+merge time was probably 3-4 hours). Hard to compare like-for-like with round 3 (~78 min) because the session gap added a lot of dead time.

## What worked

- **Parallel-via-manual-worktrees** remains the right pattern. Per-module file scope keeps merge conflicts trivial (only entry `openapi.yaml` gets non-overlapping appends).
- **2 of 4 PRs landed green first-try.** That's a real measure of agent self-correctness — the four new gotchas from rounds 1-3 (OpenAPI 3.0 nullable, `Page<>` shape, `replaceChildren` unique-index trap, varchar widths) are paying back compounding interest.
- **Pre-existing CI flake / gap recovery worked**: I lost two agent reply messages during a long session gap, but each worktree still had complete code (commits 19/37/24/25 files). Pushed each as-is, let CI judge. Round 4's biggest lesson: **the worktree itself is the source of truth, not the agent's text reply.**

## What broke (and what to learn)

### Gotcha #1: Agent skipped `spotless:apply`

The household-01d agent returned a TERSE final report ("All expected files. Now waiting for verify.") — looked like it was cut off mid-thought. It didn't run `spotless:apply`. CI caught it. Parent ran `mvnw spotless:apply` and pushed. Cost: one extra CI roundtrip (~6 min).

**Fix for `templates/agent-prompt-template.md`**: explicitly state in the verify-loop section that `spotless:apply` MUST run after verify is green, NOT as an optional cleanup. Reinforce this in the "What to report back" criteria.

### Gotcha #2: YAML inline-flow style breaks on commas in unquoted strings

The household-01d ticket-writer wrote one OpenAPI response description with an internal comma:

```yaml
'409': { description: Target already in a household, or primary collision, content: { ... } }
```

In YAML's `{ ... }` flow style, commas separate map entries. The unquoted string `Target already in a household, or primary collision` gets parsed as two entries:
- `description: Target already in a household` (truncated)
- `or primary collision`: (no value — broken)

swagger-cli then rejects the response because it doesn't match the Response Object schema. Parent had to quote the description string.

**Fix for `templates/agent-prompt-template.md`**: add to the gotcha list — "When using YAML inline-flow `{ ... }` for an OpenAPI response object, ANY description containing a comma, colon, or quote MUST be single-quoted. Safer default: multi-line block style for any description longer than 3 words."

### Gotcha #3: MockMvc `put("...%20...")` double-encodes URL paths; Spring's `StrictHttpFirewall` rejects

The nutrition-01d IT did:

```java
mvc.perform(put("/api/v1/nutrition/ingredients/chicken%20breast/correction")...)
```

MockMvc treats the string as a literal URL template, sees the `%` and re-encodes it to `%25`, producing `chicken%2520breast`. Spring's `StrictHttpFirewall` blocks URLs containing `%25` in path segments. Routing fails → 400 with no handler matched.

Three workarounds attempted (URI template form, `URI.create()`, both still double-encoded). What actually worked: change the test fixture's search term to one with no space (`chicken-breast`), sidestepping URL encoding entirely. The normaliser logic (lowercase + collapse-whitespace) is exercised by `JournalServiceTest` instead.

**Fix for `templates/agent-prompt-template.md`**: add gotcha — "For path-variable IT tests, use search terms / IDs that don't require URL encoding. Strict firewall + MockMvc auto-encoding combine to reject `%XX`-containing URLs even when they decode correctly. Move encoding-sensitive coverage to unit tests on the normaliser/decoder, not the HTTP layer."

## Round-on-round comparison (cumulative)

| Round | Style | Wall-clock | Agent ver iter avg | Parent fixes avg |
|---|---|---|---|---|
| 1 (4 modules) | serial | ~4 h | 2 | 1.25 |
| 2 (4 modules) | serial | ~5.5 h | 1.25 | 0 |
| 3 (4 modules) | parallel | ~1.3 h | 2-5 (under contention) | 0 |
| 4 (4 modules) | parallel | ~3-4 h (uneven, session gap) | uncaptured | 0.75 |

Round 4's parent-fix count rose from 0 (rounds 2+3) back to 0.75 — driven by edge-cases the agent prompt hadn't preemptively warned against (Spotless, YAML commas, URL encoding). Each is now in the gotcha list. Round 5 should be back to 0 if these are stable lessons.

## Implications for ai-workflow templates

1. **`templates/agent-prompt-template.md`** — add the 3 round-4 gotchas listed above.
2. **`playbook/parallel-agent-coordination.md`** — add a note: "Agent reply messages can time out if the parent session is paused. The worktree is the source of truth — push it and let CI judge."
3. **`templates/ticket-template.md`** — no changes; the ticket-writer correctly used the new Page<> + nullable patterns, and didn't repeat the round-3 issues.

## What's next

Per the round-1+2+3 deferral maps, the remaining Wave 2 sub-tickets are:

- household: 01e merge (blocked on preference soft-prefs ticket — could trigger a preference-01c ticket first if you want this unblocked)
- nutrition: 01e directives, 01f calc service, 01g floor-gate, 01h aggregation
- provisions: 01e waste, 01f planner bundle, 01g cook events, 01h grocery import, 01i staples, 01j batch-cook, 01k expiry, 01l feedback, 01m pantry gate
- recipe: 01e substitutions, 01f write API, 01g promotion/archive, 01h embeddings, 01i search, 01j helpers, 01k AI tag inference

That's ~18 sub-tickets left in Wave 2 alone. At the round-3/4 pace (~80 min per round of 4), that's ~6 hours of active orchestration. Then Wave 3 (orchestrators — planner, discovery, feedback module).

The pattern is mature. The user could AFK on the next 4-5 rounds if they want.

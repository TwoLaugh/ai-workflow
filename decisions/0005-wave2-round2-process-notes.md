# Decision 0005 — Wave 2 round 2 process notes

**Date**: 2026-05-10
**Status**: Round 2 complete — 4 implementation tickets shipped to main
**Source**: TwoLaugh/foodSystem PRs #15–#18

## What landed

| Ticket | PR | Files | Agent runtime | Verify iter (agent / parent) | Wall-clock per ticket |
|---|---|---|---|---|---|
| provisions-01b | [#15](https://github.com/TwoLaugh/foodSystem/pull/15) | ~26 | ~14 min | 0 (denied) / 1 first try | ~25 min |
| household-01b | [#16](https://github.com/TwoLaugh/foodSystem/pull/16) | ~32 | ~18 min | 1 / 1 (confirmation) | ~30 min |
| recipe-01b | [#17](https://github.com/TwoLaugh/foodSystem/pull/17) | ~30 | ~29 min | 2 / 1 | ~40 min |
| nutrition-01b | [#18](https://github.com/TwoLaugh/foodSystem/pull/18) | ~49 | ~30 min | 2 / 1 | ~40 min |

**Round-2 total wall-clock**: 23:45 → 05:12 ≈ ~5h25m elapsed (overnight, with breaks). Active orchestration time substantially less.

**Total project**: 8 PRs in <24h, 18 modules' worth of code, ~430 tests passing.

## Round 2 vs Round 1 — was it quicker?

**Per-ticket pace was comparable**: round 1 averaged ~32 min/ticket wall-clock, round 2 averaged ~34 min/ticket. Surface-level no faster.

**But the *quality* of agent output improved markedly**:
- Round 1: 8 verify iterations across 4 tickets (avg 2/ticket); 5 parent-side fixes for OpenAPI 3.0 schema gotchas + the LLD spec error.
- Round 2: 5 verify iterations across 4 tickets (avg 1.25/ticket); 0 parent-side fixes after the agent's pass — every parent verify was a single confirmation run.

That's the gotcha-list paying back. The four new gotchas added to `templates/agent-prompt-template.md` after round 1 (OpenAPI nullable+$ref, allOf+nullable, Page<> additionalProperties, replaceChildren+unique-index hazard, varchar widths) prevented those exact bug classes in round 2. The recipe-01b agent caught the ArchUnit `springWebStaysInApi` violation itself in iter 1 (RestClient outside `..api../..config..`); that's a class of bug the agent now self-catches because the rule is mature.

## What worked

- **Per-module file scope held up.** All four agents appended cleanly to their existing 01a files (exception handler, paths/schemas yaml) without breaking sibling modules. The merge-zone refactor's value compounds with each additional ticket.
- **Self-catching by the agent**: recipe-01b caught its own ArchUnit violation in iter 1 and fixed it in iter 2. Nutrition-01b caught its own Boolean-null vs schema mismatch and fixed it in iter 2. Household-01b's 1-iter pass was driven by the prompt's explicit gotcha-list reminders.
- **`UpsertResult<T>` invention** (provisions-01b) for the 201/200 PUT split. The ticket didn't spec the mechanism; the agent picked a sensible pattern and documented it. This is the kind of judgment call agents now make competently when the prompt is well-grounded.
- **Recipe-01b's `currentVersionBody` vs `currentVersionDetails`** — the agent noticed a ticket-vs-existing-code mismatch and chose to preserve backwards-compat with the round-1 IT. Good taste.

## What didn't work / is still flaky

- **Agent shell access still sporadic**: provisions-01b had Bash/PowerShell denied (same as round 1's 50% denial rate). Parent verify covered, no real cost.
- **Worktree isolation never came back**. Both rounds ran serial. Total active orchestration is ~50% latency-bound (waiting on agent / mvn / CI). True parallel would cut wall-clock by ~3-4×.
- **Ticket-writer agent invented the 4-page nested `page: { number, size }` schema shape** for two of the four 01b tickets, despite round-1 having established the flat shape. Two implementation agents had to flag this and override (one for nutrition, one for household). Fix for future ticket-writers: explicitly seed the canonical Page<> example in the ticket-writer prompt. Not currently in `templates/ticket-template.md`.

## Architectural insight that compounds

The nutrition-01a's `replaceChildren()` + unique-index trap from round 1 was load-bearing for round 2. **Every** new aggregate in nutrition-01b had `(parent_id, business_key)` style unique constraints (intake_slot, intake_snack, intake_audit). Without the round-1 lesson, this would have been a 30-45 min iteration sink in round 2's biggest ticket. The agent applied "compute change-set BEFORE mutating" as a default — flagged in the prompt, internalised correctly.

This is the model: **lessons from one round become defaults for the next**. Round 2's clean parent-verify-on-first-try track record validates this.

## Implications for ai-workflow templates

Patches needed (small):

1. **`templates/ticket-template.md`** — add a "Spring Page<T> schema shape" canonical example so ticket-writer agents copy the flat form, not invent nested forms:
   ```yaml
   FooDtoPage:
     type: object
     additionalProperties: true   # tolerate Spring's pageable/sort
     required: [content, totalElements, totalPages, number, size]
     properties:
       content: { type: array, items: { $ref: ... } }
       totalElements: { type: integer, format: int64 }
       totalPages: { type: integer }
       number: { type: integer }
       size: { type: integer }
   ```
2. **`templates/agent-prompt-template.md` — already contains** ArchUnit `springWebStaysInApi` info implicitly via the round-1 boundary test, but worth making explicit: HTTP-client adapters (RestClient/WebClient) must live in `..api..` or `..config..`, NOT in `domain.service.internal`. The recipe-01b iter 1 was a cheap discovery; preventing it in the prompt would save ~3-4 min next time.

Both patches lift round 3+ pace incrementally.

## Next-step recommendations

1. **Round 3 is plannable now**: each module has a clear sub-ticket map (household-01c invites + 01d merge; nutrition-01c journal + 01d ingredient-mapping; provisions-01c budget + 01d supplier; recipe-01c manual-edit + 01d branch-creation). 8 tickets — could split across two rounds.
2. **Attempt worktree isolation at next session start.** A fresh session may re-detect git state. If it works, round 3 could finish in ~50 min wall-clock instead of ~2.5h.
3. **The pattern is mature enough that the user could now AFK on round 3** if worktree works. Each ticket is well-grounded; agents are self-correcting; parent intervention is light.

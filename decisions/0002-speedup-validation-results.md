# Decision 0002 — Speedup validation results

**Date**: 2026-05-09
**Status**: Validated
**Source**: TwoLaugh/foodSystem `preference-01b` ticket (post-fix), compared against `preference-01a` baseline

## What we measured

Decision 0001 ranked the per-ticket time costs and proposed mitigations. This doc reports the actual delta after applying them. Same-shape ticket (`preference-01b` — filter service + allergen derivatives, comparable scope to `preference-01a`'s aggregate + endpoints).

## Changes applied between the two runs

1. **`docker-java api.version=1.44` pin** — unblocks Testcontainers ITs on Windows / Docker Desktop. Now agents AND parent run `mvn verify` to completion locally.
2. **Surefire `forkCount=1C reuseForks=true`** — parallelises unit tests across N JVMs.
3. **Pitest off PR runs** — only triggers on push-to-main and PRs labelled `mutation`. Saves 5-7 min CI watch per PR.
4. **Verbatim shape snippets in agent prompt** — replaced "see auth/AuthServiceImpl.java for shape" with the actual ~10-line code excerpts inline. Ticket `01b` prompt was ~200 lines longer; agent prompt cost up, file-read tool calls down.

## Numbers

| Metric | preference-01a (baseline) | preference-01b (post-fix) | Δ |
|---|---|---|---|
| Agent runtime (wall-clock) | ~25 min | **~9 min** | **-65%** |
| Tool calls total | 100+ | 43 | -55% |
| File-read tool calls | ~25-30 | 17 | -40% |
| Verify iterations agent-side | 1 | 1 | (same) |
| Parent CI roundtrips after agent done | 3 (Hibernate-runtime fixes) | 0 | **gone** |
| CI build-test-lint duration | ~3 min | ~3:14 | (same) |
| CI Pitest wait | ~5-7 min (gated) | skipped (no label) | **-5-7 min per PR** |
| Total elapsed including parent verify + merge | ~70 min | **~25 min** | **-65%** |

## What drove most of the saving

In rough rank order:

1. **Docker bridge fixed (api.version pin)**: closed the entire 3-CI-roundtrip loop where the agent skipped ITs and the parent had to fix Hibernate-runtime issues from CI feedback. Saved ~30 min per ticket. **Single biggest win.**
2. **Verbatim shape snippets in the prompt**: cut agent runtime from 25 min → 9 min. The agent stopped reading files to find conventions because the conventions were already inlined. Saved ~16 min per ticket.
3. **Pitest off PR runs**: saved 5-7 min of CI watch per PR. Smaller but reliable.
4. **Surefire parallel forks**: minor (~30s) on this codebase. Bigger payoff on larger unit-test suites.

## What didn't move the needle (yet)

- **Verify iterations**: still 1 agent-side. The fix-loop wasn't the bottleneck; the bigger problem was the agent skipping it entirely. Once it ran end-to-end, tickets converged on the first try.
- **CI build duration**: still ~3 min. Unaffected by these changes; would need IT-base-class refactor or larger CI runner.

## What's the next bottleneck?

With the easy wins applied, the remaining time per ticket breaks down roughly:

| Cost | Time | Note |
|---|---|---|
| Agent runtime (1 verify pass + write code) | 8-12 min | Mostly Spring context startup × multiple ITs. Hard to compress further without IT-base-class refactor. |
| Parent verify + commit + push + PR + merge | 8-12 min | Mostly mvn-verify + diff review + manual gh CLI dance. |
| CI watch | 3-4 min | Build-test-lint only; Pitest skipped. Could parallelise with other work. |
| **Total typical-case** | **~25 min** | Down from ~70 min. |

Next-tier optimisations (all from decision-0001's P1/P2):

- **IT shared base class** (~1-2 min per verify — biggest remaining ticket cost). 1-2 hr refactor.
- **Pre-warm Docker images on agent startup** (~10s once per ticket). Trivial; per-prompt addition.
- **Cap Pitest target packages per-ticket on push-to-main** (~3-5 min per CI). Per-ticket spec discipline.

The IT base class refactor probably gets us to **~20 min average per ticket** if all goes well. That's the soft floor without paid CI runners.

## Caveats — single sample, biased ticket

`preference-01b` is a particularly easy ticket: read-only service, well-defined data shape, no cross-cutting changes. The ~25 min number is best-case.

A harder ticket with auth interactions, novel patterns the LLD doesn't fully spec, or 3-way parallel agents would still take ~45-60 min. Multi-sample data needed before claiming this as the steady-state pace.

That said: even the hard cases now don't have the **3-CI-roundtrip Hibernate-fix loop** in them, so the worst-case has compressed from ~90 min → ~60 min.

## Implications for ai-workflow's templates

- `templates/agent-prompt-template.md`'s "shape snippets inline" guidance is validated. Apply by default; not an optional optimisation.
- `playbook/verify-loop.md` should stress that the loop only works if Docker is working agent-side. The `docker-java.properties` pin (or its equivalent for other stacks) is a pre-requisite, not a "nice to have".
- `decisions/0001`'s P0 mitigations are confirmed. Move to P1 work next: IT shared base class.

## Open questions

- Will the speedup hold across 3-way parallel batches? `auth-01b/c` parallel had high merge-conflict overhead; haven't re-run that pattern post-fix to see if it shrinks.
- Will the agent runtime regress if we extend the prompt with even more shape snippets? At what point is the prompt cost > the latency saving? Probably worth instrumenting.
- Is the ~9-min agent runtime mostly latency-bound (LLM + mvn) or thinking-bound (LLM compute)? Different optimisations apply.

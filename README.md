# ai-workflow

A ticketed agent-driven SWE workflow extracted from the MealPrep AI project. Reusable across complex projects; iterates independently of any one codebase.

## What this is

A working system for getting AI agents to ship production-quality code on projects too big for one human to hand-code. Born out of a real project (a personalised meal-planning system, ~70-90 backend tickets across 4 waves) where:

- The codebase is too large to fit in one agent's context.
- One human can't hand-code it in a reasonable timeframe.
- Letting an unsupervised "build the whole thing" agent loose produces unreviewable garbage.
- Going purely solo gives up the productivity multiplier.

The workflow that emerged: **ticket-sized agents, self-verifying loops, a curated playbook codifying conventions, and a human acting as architect + reviewer + merge orchestrator.**

This repo is the artefacts. Not the project that produced them; just the reusable pattern.

## What's here

```
playbook/                     # the rules of engagement — what's a ticket, what's a self-verify, what's a review
templates/                    # ticket / agent-prompt / style-guide skeletons
conventions/                  # specific patterns proven in the field (verify loops, parallel agents, etc.)
decisions/                    # decision log — every non-obvious choice + why + what we tried
starter-kits/                 # bootstrap projects per stack (Spring Boot first; more later)
```

## Status

- **Source project**: [TwoLaugh/foodSystem](https://github.com/TwoLaugh/foodSystem) — the MealPrep AI codebase. ~9 production tickets shipped at extraction time.
- **First extraction commit**: 2026-05-08 — playbook + templates extracted from the project's `lld/implementation-playbook.md` and ticket files.
- **Validation status**: pattern proven on Wave 1 (project-setup, core, auth, ai modules). Wave 2+ pending; this repo evolves alongside.

## When to use this

If you're building something where:

- The total scope is more than ~10 tickets / 50 hours of solo work.
- You want AI to do the majority of the typing but you to control the architecture.
- You care about test quality, not just "tests exist".
- You have time to write specs upfront (LLDs / detailed tickets) before implementation.

Don't use this for:

- One-off scripts or prototypes — too much overhead.
- Codebases without a clear architectural shape — the playbook needs an architecture to enforce.
- Throwaway research code — formality is a tax you don't need to pay.

## How to use

The intended flow for a new project:

1. **Write HLD + LLD** — same way you would without AI. The agent reads these as the source of truth. (Process docs for HLD/LLD design are out of scope for this repo; bring your own.)
2. **Bootstrap the codebase** — copy a starter kit if one matches your stack; otherwise hand-build the bootstrap (pom/CI/conventions/test infrastructure). The first ticket is "project-setup" and lands the conventions + tooling.
3. **Write tickets per `templates/ticket-template.md`** — concise, link-heavy, ~10-25 file scope.
4. **Spawn agents per `templates/agent-prompt-template.md`** — they read the ticket + LLD + playbook, write code, self-verify with `mvn verify` (or your stack's equivalent), report back.
5. **Verify + merge** — you review the report, eyeball the diff, run CI, merge.
6. **Iterate the playbook** — when something hurts, fix the conventions here and update.

The expectation is **30-60 minutes per ticket** when the loop is well-tuned, including 5-10 min of human-in-the-loop time per ticket. See `decisions/0001-pacing-and-bottlenecks.md` for what actually drives that.

## Philosophy

The biggest lessons from the source project:

1. **Tickets must be agent-sized.** The original `auth-01` ticket was 27 spec items / 40 files — the agent shipped a workable implementation but the human review was unmanageable. We split it into `auth-01a/b/c` (~12 files each); ship velocity tripled.
2. **Self-verify or it didn't ship.** The agent runs `mvn verify` (full unit + IT) and iterates until green BEFORE reporting back. Otherwise the human becomes a manual fix loop and the speedup vanishes.
3. **Parallel agents need scope walls in their prompts.** When `01b` and `01c` run in parallel, each prompt explicitly names the other's scope to avoid file collisions. Even then, expect 5-15 min of merge-conflict resolution per parallel batch.
4. **The LLD is the single source of truth.** Tickets reference the LLD; agents read the LLD; reviewer trusts the LLD. If the LLD is wrong, fix it before writing more tickets.
5. **Frontend coupling matters.** Backend-first works, but expect rework when the UI hits real users. Ship screen-by-screen if the goal is dogfooding; ship backend-first only when the modules are mostly internal.

See `decisions/` for the full decision log behind these.

## License

To be added (likely MIT).

# Decision 0012 — Wave 3 (orchestrators): rounds 1–4 + the gotcha cluster

**Date**: 2026-05-17
**Status**: Wave 3 rounds 1–4 complete — 4 modules (planner, discovery, feedback, adaptation-pipeline) at tier `01d`. ~18 PRs merged.
**Source**: TwoLaugh/fdssys256 (renamed from foodSystem mid-wave — see §Process/billing) PRs #41–#58

## What landed

Wave 3 is the orchestrator wave: `planner` (meal-plan composition), `discovery` (recipe sourcing), `feedback` (classify+route), `adaptation-pipeline` (the optimiser). 30 tickets were written up-front by 4 parallel ticket-writing agents (one per module LLD), then implemented in tiered rounds:

| Round | Tickets | Shape |
|---|---|---|
| 1 | 4× `01a` foundations + 1 migration-collision fix | 3-way parallel (planner held to 1b) |
| 2 | 4× `01b`/`01c` (contracts, services, lifecycle, query) | 3-way + 2-way batches |
| 3 | adaptation/discovery/feedback `01c` (worker, source SPI, classifier) | 3-way parallel |
| 4 | 4× `01d` (beam search, triggers+listeners, job runner, router) | 4-way parallel |

~14 tickets remain (planner `01e–01l`, others `01e–01f`).

## The headline lesson: recurring wave-2 gotchas are now *proactive* checklist items, not reactive fixes

Wave 3 re-tripped **every** major wave-2 retro item at least once, despite them being in the prompt template. The cost was enormous — multiple PRs went through 4–6 CI cycles each. The pattern: a fresh agent reads the gotcha list but doesn't connect an abstract rule to its concrete situation until CI fails. **Rules in a prompt are necessary but not sufficient; the agent needs the rule applied to its ticket's specifics.**

### The StaleObjectStateException / native-UPDATE cluster (round-8 wave-2 retro, recurred 3×)

`@Version` entity + an async/`AFTER_COMMIT` path that loads→mutates→`saveAndFlush` → `StaleObjectStateException` because the publisher's persistence context still holds the row. Recurred in: adaptation `PendingChange` supersession IT, feedback `FeedbackClassificationListener`, discovery cancel flow. The fix (native `@Modifying @Query` UPDATE) has **four sub-traps**, each of which cost a separate CI cycle:

1. **JPQL must reference the entity FIELD name** (`e.optimisticVersion`), not `e.version` or the column name → `UnknownPathException` fails context-load for *every* IT in the project.
2. **Bump the version column in the SQL** (`SET ..., e.optimisticVersion = e.optimisticVersion + 1`) so subsequent JPA reads don't see a stale version.
3. **Unit-test mocks of the new `@Modifying` method** must `.thenReturn(1)` AND (if downstream asserts entity state) `.thenAnswer(...)` to simulate the mutation — Mockito returns `0` by default → "rows == 0 → throw NotFound".
4. **`@Modifying(clearAutomatically = true, flushAutomatically = true)`** is required when the same request flow loaded the entity *before* the bulk UPDATE and re-reads it *after* (e.g., a controller that calls `cancelJob` then re-fetches for the response DTO) — otherwise the re-read returns the stale first-level-cached pre-update entity.

→ **New prompt-template rule**: "If a ticket has an `@Version` entity touched by an async/AFTER_COMMIT/cross-request path, use a native `@Modifying(clearAutomatically=true, flushAutomatically=true) @Query` from the start. Reference the JPA field name. Bump the version column. Stub `.thenReturn(1)` in unit tests."

### `@TransactionalEventListener(AFTER_COMMIT)` silently drops events published outside an active tx

feedback's classifier published `FeedbackProcessedEvent` *after* its `TransactionTemplate.executeWithoutResult` block returned. With `fallbackExecution=false` (default) and no active tx, Spring **silently discards** the event — the test's `@TransactionalEventListener`-based capture saw nothing and timed out. Fix: publish INSIDE the tx body. This is distinct from the round-7 propagation rule and worth its own line.

### Time-bomb tests (the most expensive single failure of the wave)

`HouseholdInvitesServiceTest` hardcoded `fixedNow = Instant.parse("2026-05-09T12:00:00Z")`. `HouseholdInviteMapper.deriveStatus` correctly compares `expiresAt` against real wall-clock `Instant.now()` (production-correct — an invite genuinely expires in real time). Invites created at `fixedNow + 7d` (2026-05-16) silently rotted to `EXPIRED` once the real date rolled past — failing `createInvite_byPrimary...` **identically on every branch**, in a module nobody in wave 3 had touched. It masqueraded as a per-PR failure and was diagnosed three times across #54/#56/#57 before the penny dropped.

→ **Two new rules**: (a) test fixtures with relative-expiry semantics must anchor `now` to real wall-clock (`Instant.now().truncatedTo(DAYS).plus(1, DAYS)`), never a hardcoded date, when the code-under-test compares against real `Instant.now()`. (b) **A test failing identically across all parallel branches is a main-branch problem — check main first** before diagnosing per-PR. This would have saved ~3 diagnosis cycles.

### Other recurrences (each cost ≥1 CI cycle)

- **Migration-timestamp collision**: discovery-01a and adaptation-01a independently picked `V20260615120000`. Each branch's CI passed (only saw its own); the collision only surfaced once both were on main. → Pre-assign each module a distinct migration hour-range in the ticket.
- **`@ConditionalOnMissingBean` on a `@Component` class** (round-5 wave-2 retro): InHouseRobotsTxtGate gated *itself* off because the conditional fires during component-scan before any alternative bean registers. Plain `@Component`; use `@Configuration + @Bean @ConditionalOnMissingBean` for the override case.
- **Test classpath shadows main `application.properties`** (round-1 wave-2 retro): planner's `@ConfigurationProperties` validation failed under `@SpringBootTest` because `src/test/resources/application.properties` shadows main. Mirror keys in both.
- **Spotless `:apply` skipped**: still recurring despite being a hard rule.

### New gotchas unique to wave 3

- **`@AssertTrue`/`@AssertFalse` validator methods on request records are serialized by Jackson** as boolean JSON properties (`isMaxRecipesPerSourceWithinTotal()` → `"maxRecipesPerSourceWithinTotal": false`), which strict OpenAPI request-body validators reject as an unexpected property. → Annotate the validator method `@JsonIgnore`.
- **`MissingServletRequestParameterException` → 500, not 400**, unless the module's `@RestControllerAdvice` (which is `@Order(HIGHEST_PRECEDENCE)`) explicitly maps it. Spring's default 400 handling is overridden by the project-wide catch-all.
- **Multi-constructor Spring bean with no `@Autowired`**: a prod constructor + a test-only constructor (neither annotated) → `"No default constructor found"`. Annotate the production constructor `@Autowired`.
- **Shared test fixtures must satisfy cross-field validators**: `DiscoveryTestData.sampleConstraints()` had `maxRecipesPerSource=20` while ITs passed `requestedCount=5`, tripping a cross-field `@AssertTrue` → every POST 400'd. A shared fixture is consumed by many tests; it must be valid against the *whole* validation graph.
- **A live async runner races IT-seeded state**: once discovery-01d wired the real `DiscoveryJobRunner`, ITs that seeded a job *via the POST endpoint* then asserted the seeded state failed — the runner moved the job before the assertion. State-contract tests must seed **directly via the repository** (no POST → no event → runner never touches it). The `@Scheduled` orphan sweep was safe only because of its 5-min initial delay.

## Process lessons

### GitHub Actions billing is a hard wall — make the repo public

A private repo with the default **$0 spending limit** hard-stops CI the moment the free monthly minutes are exhausted ("the job was not started because recent account payments have failed"). At wave velocity (60–100+ CI runs/wave × ~12 min) the 2k/mo private allotment evaporates. **Public repos get unlimited free Actions minutes.** The user renamed `foodSystem`→`fdssys256` and flipped it public; CI resumed instantly at zero cost. A self-hosted runner on the dev laptop was rejected: 7.7 GB RAM shared with Windows+Docker+IDE can't reliably host a Testcontainers `mvn verify` (we'd already seen OOM/Docker-500 flakes there). Correct self-hosted answer would be a cheap dedicated cloud Linux box, not the laptop. **Bake into the playbook: if the project is CI-heavy and not proprietary, make the repo public on day one.**

### The autonomous `ScheduleWakeup` loop did not fire reliably

Repeated `ScheduleWakeup` calls to poll CI did **not** auto-resume the session in this harness — every resume correlated with a user message. CI has no callback into the session, so blind timers added dead-air after CI finished. **The reliable mechanism is background-bash completion notifications**: a backgrounded `until grep -q '<done>' ...; do sleep N; done` poll loop emits a real `<task-notification>` on exit. Use that for CI-watching, not `ScheduleWakeup`. (Local-agent and background-bash completion notifications worked reliably all wave.)

### Other process notes

- **0-byte agent transcript ≠ stuck.** 4 round-1 agents were prematurely `TaskStop`'d because their JSONL output files hadn't flushed; the kill summaries revealed they were mid-implementation. Never infer agent state from output-file size; wait for the completion notification. (Cost: a full round-1 re-spawn.)
- **Worktree is source of truth on rate-limit/timeout.** Ticket-writing agents that hit the daily cap returned an error string but had already written all ticket files to disk. Inspect the worktree, don't trust the reply.
- **4-way parallel is viable** on a public repo with CI as the gate (vs round-3 wave-2's "3 is steady-state" — that was Docker-on-Windows constrained; with cloud CI the constraint is gone). Round 4 ran 4 agents cleanly.
- **Cross-module naming reconciliation up-front works.** `WAVE3-NAMING-RECONCILIATION.md` (OptimiserService→AdaptationService mapping, written before implementation) meant planner-01j/feedback-01d agents had a single source of truth for the contract; zero naming-drift fixes needed.

## Net

Wave-3 rounds 1–4 shipped 4 orchestrator modules through `01d`. Velocity was gated almost entirely by **recurring known gotchas re-tripped by fresh agents** + the **CI billing wall** + the **unreliable wakeup loop** — not by genuine design difficulty. The fixes (proactive native-UPDATE recipe, public repo, background-poll CI watch, time-bomb-test rule, "identical-failure-across-branches = main bug" heuristic) should compress rounds 5+ substantially. The decision-log → prompt-template feedback loop remains the highest-leverage process investment.

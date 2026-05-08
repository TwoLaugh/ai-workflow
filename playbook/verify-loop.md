# The verify loop

The single most important convention in the workflow. **The agent runs the full test suite and iterates on failures BEFORE reporting back to you.** Otherwise you become the manual fix loop and the speedup vanishes.

## What "verify" means

For a Spring Boot + Maven + Postgres stack:

```
1. ./mvnw -Dspotless.check.skip=true verify
2. On failures: read errors, fix smallest possible thing, goto 1
3. Cap at 5 iterations; STOP and report if still red after 5
4. ./mvnw spotless:apply
5. ./mvnw spotless:check
```

`mvn verify` runs:
- Compile (catches syntax + visibility issues)
- Surefire (unit tests)
- Failsafe (Testcontainers ITs — needs Docker)
- ArchUnit rules (architectural boundary enforcement)
- Coverage gates (JaCoCo) and mutation score (Pitest, if configured)

## Why ALL of it, not just unit tests

Unit tests with mocked repos miss:

- Hibernate-runtime issues (lazy loading, fetch joins, JSONB roundtrips, optimistic-lock flush timing)
- Spring Boot context-load failures (DI ambiguity, missing beans, profile mismatches)
- ArchUnit boundary violations
- OpenAPI spec drift
- Migration / schema mismatches

These are 30-50% of real bugs in our experience. If the agent skips ITs, those bugs surface in CI and the fix loop becomes synchronous + multi-cycle. Cost: typically 3 CI roundtrips × 5 min = 15 min of pure waiting per ticket. Versus running locally inside the agent: ~90 seconds of `mvn verify` re-runs.

## What the agent should do on failure

For each iteration:

1. **Read the error**, not just "did the build go red". Failsafe reports show stack traces; the agent should grep for the actual message + the test method name.
2. **Fix the smallest possible thing.** Resist the urge to refactor. The first fix should be a one-liner if possible. Add a missing import. Change `save` to `saveAndFlush`. Add a column constraint.
3. **Re-run.** If green: move to spotless. If still red: did the fix make things worse, or expose a deeper issue? Log the iteration explicitly in the agent's report.

After 5 iterations: STOP. The agent reports what it tried and what's still failing. Better to surface a stuck loop than ship 4 bad attempts at the same fix.

## Common gotchas the loop catches

- **Compile errors** from missing imports / visibility. Caught immediately.
- **Spring DI**: `Cannot find bean of type X` → typically a missing `@Component` / `@Service` annotation or a constructor-ambiguity. Agent fixes by reading the error.
- **`MultipleBagFetchException`** when `@EntityGraph` fetches multiple `@OneToMany List<>` — drop the EntityGraph or convert to `Set<>`.
- **`@Version` flush timing**: PUT response has stale version → use `saveAndFlush`.
- **`LazyInitializationException`** in test code that reads lazy collections outside its tx — fix is usually re-fetch with explicit fetch join, or wrap test in `@Transactional`.
- **`text[]` Postgres array mapping**: Hibernate's `ListArrayType` is brittle on Spring Boot 3.2 / hibernate-utils-63; use `jsonb List<String>` via `@Type(JsonBinaryType.class)`.
- **CRLF line endings on Windows** breaking regex parsers — use `\R` not `\n`.

## What the loop CAN'T catch

- Wrong invariants. If the spec says "no audit row on no-op PUT" and the implementation
  writes one, but the test doesn't check, the loop says green. Mutation testing helps but
  isn't a substitute for the human reviewer eyeballing the test set.
- Architectural drift that ArchUnit doesn't model. New patterns the rules don't cover slip through.
- Performance regressions. The verify loop times each test but doesn't gate on perf budget.
  k6 / JMH scripts (if any) run separately.

## Pre-flight: the parent's job before kicking off the agent

Before launching the agent, the parent confirms:

```
1. main is green on origin (no broken predecessor sneaking in)
2. ./mvnw -DskipITs test passes locally on main
3. Docker daemon is running (or accept that ITs will skip on the agent's side)
```

If pre-flight is red, fix it first. Don't pile bugs on bugs.

## Post-flight: the parent's job after the agent reports done

After the agent's report:

```
1. cd into the agent's worktree
2. git status — confirm changes match the report
3. ./mvnw -Dspotless.check.skip=true verify — re-run the loop yourself
4. eyeball the diff for hollowness / scope creep / missing tests
5. commit, push, open PR, watch CI, merge
```

The parent verify on step 3 is fast (~3-8 min) because the agent already converged.
Most of the parent's time is in steps 4-5.

## Pacing

For a typical ticket on a Spring Boot + Postgres stack:

| Phase | Time |
|---|---|
| Agent writing code | 5-10 min |
| Agent verify loop (1-3 iterations × 3-5 min mvn) | 5-15 min |
| Agent reporting | 1-2 min |
| Parent verify | 3-8 min |
| Parent diff review + commit + push + PR | 5-15 min |
| CI watch (build 3 min + Pitest 5-7 min if gated) | 7-10 min |
| Merge | 1 min |

Total: ~30-60 min per ticket. Most of the variance is in the agent's iteration count
(usually 1-2 iterations; occasional 4-5 means a deeper miss in the prompt or LLD).

See `decisions/0001-pacing-and-bottlenecks.md` for the full breakdown of where time goes
and how to compress it.

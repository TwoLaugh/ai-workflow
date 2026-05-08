# Decision 0001 — Pacing and bottlenecks

**Date**: 2026-05-08
**Status**: Active
**Source**: MealPrep AI project, end of Wave 1 (~9 tickets shipped)

## Where time actually goes per ticket

Measured across the auth-01 / preference-01a / ai-01a-d tickets. Numbers are typical-case, not best-case.

| Cost | Time per ticket | Cumulative for 70 tickets |
|---|---|---|
| Spring Boot test context startup × N IT classes | 3-5 min | 4-6 hours |
| Agent LLM round-trip latency (100-120 tool calls × 1-3s) | 8-12 min | 9-14 hours |
| BCrypt-bound IT tests (auth timing-parity + throttle) | ~40s | wash (one-time per affected ticket) |
| Parallel-agent merge resolution | 5-30 min (high variance) | 5-15 hours |
| CI watch (build 3min + Pitest 5-7min, sequential) | 7-10 min | 8-12 hours |
| Windows-specific overhead (Docker WSL2, NTFS) | 2-3 min | 2-3 hours |
| Conflict-resolution iteration when fixes break sibling tests | 5-15 min (when triggered) | unpredictable |

**Per-ticket realistic average**: 30-45 min (single agent), or ~25-30 min effective when run as parallel batch.

## What dominates

### 1. Spring Boot test infrastructure (~30% of total)

Each `@SpringBootTest` IT class spins up a fresh Spring context (10-30s). Spring caches when configurations match exactly, but `@Import(...)` variations + `@MockBean` + profile differences cause cache misses. By Wave 1 end we had ~10 IT classes; many fresh contexts per `mvn verify`.

**Mitigations:**
- `surefire/failsafe forkCount=2C` for parallel test execution
- Shared IT base class to reduce `@Import` variation and improve context reuse
- `@DataJpaTest` / `@WebMvcTest` for slice-only ITs that don't need full context

### 2. Agent LLM round-trip latency (~25% of total)

Most agent tool calls are file reads. A typical agent reads ~30 source files to find shape references. Each read is a 1-3s API round-trip.

**Mitigations:**
- Pre-load relevant snippets into the agent's prompt (verbatim file excerpts under "Read these first")
- Use Anthropic prompt caching for stable LLD/playbook content
- Cap agent's tool budget per task; surface "I need more context" explicitly

### 3. Parallel-agent merge resolution (~20% of total, high variance)

When 3 sibling tickets run in parallel and all touch shared files (controller, exception handler, properties, OpenAPI), the human spends 5-30 min hand-merging.

**Mitigations:**
- Size sibling tickets so they touch different files
- One ticket explicitly owns each shared file; others extend via `@ConfigurationProperties` sub-records or new methods
- Accept that perfect non-overlap is impossible; budget 10 min per sibling pair
- Pull the parallel-merge structure forward in the playbook so agents help avoid it

### 4. CI watch time (~15% of total)

`mvn verify` on CI Linux: ~3 min. Pitest: ~5-7 min. The human watches both sequentially; Pitest gates ≥70% mutation coverage which is valuable but doubles the wait.

**Mitigations:**
- Move Pitest off PR runs, only on main pushes (saves 5-7 min per PR; trade-off: mutation regressions caught one merge later)
- Or: scope Pitest to changed packages per ticket (smaller scope = faster run)
- Watch CI in background while doing other work — don't block on it

### 5. Windows-specific overhead (~5-10%)

Docker Desktop's WSL2 layer adds 3-5s per Postgres container spin. NTFS classpath scans are 10-20% slower than Linux ext4. Git Bash works fine but lacks some POSIX niceties. The `docker-java api.version` pin (see source-project's `src/test/resources/docker-java.properties`) is required.

**Mitigations:**
- WSL2 development environment recovers most of this. Half-day setup; pays back across all future tickets.
- Or: just accept the 2-3 min penalty and don't pivot.

## Mitigation priorities (low effort first)

| Priority | Change | Saves per ticket | Effort |
|---|---|---|---|
| P0 | Surefire/failsafe `forkCount=2C` parallel forks | 30-90s | 5 min |
| P0 | Move Pitest from PR-runs to main-only | 3-5 min wall-clock waiting | 1 line workflow change |
| P1 | Verbatim shape snippets in agent prompts (replace "see X" with the snippet) | 2-4 min in agent runtime | per-prompt revision; ~30 min initial |
| P1 | IT shared base class — reduce Spring context cache misses | 1-2 min per verify | 1-2 hr refactor |
| P2 | Pre-warm Docker images at agent startup | ~10s once | trivial; agent prompt addition |
| P2 | Cap Pitest target classes per-ticket | 3-5 min per CI | 1 hr config + per-ticket spec |
| P3 | Dev box pivot to WSL2 | 2-3 min per verify | half day |
| Optional | Larger CI runner (`ubuntu-latest-8core`) | 30-60s per CI | $$$ |

## What we're NOT going to do

- **Skip ITs in agent loop.** They catch the bugs unit tests miss. The cost is real but proportional to the value.
- **Disable mutation testing.** It's the only signal against hollow tests; without it, the agent's tests look fine to a reviewer but cover nothing.
- **Drop the IT-base-class refactor as too expensive.** It's a one-time cost that saves time on every subsequent ticket.

## What this implies for the workflow project

1. The `templates/agent-prompt-template.md` should embed shape-reference snippets, not just paths.
2. The `starter-kits/spring-boot-postgres/pom.xml` should ship with `forkCount=2C` and Pitest-on-main-only by default.
3. The `playbook/parallel-agent-coordination.md` convention doc should make the "scope-wall + budget 10 min for merges" expectation explicit.
4. Decision 0002 (TBD) should track WSL2 pivot if/when we do it.

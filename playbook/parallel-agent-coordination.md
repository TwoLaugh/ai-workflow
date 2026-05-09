# Parallel agent coordination

When two or more sibling tickets are independent enough to run in parallel, you can spawn
multiple agents in worktree-isolated copies of the repo and merge them sequentially. The
speedup is real (~1.6× over serial in our measurement), but less than 2× because of merge
overhead.

This doc captures what works, what doesn't, and what to expect.

## When parallel makes sense

Parallel-able when:

- Two tickets touch mostly disjoint files. Each adds a new feature stack (new entity, new
  service, new endpoint) without modifying the other's stack.
- Both depend on a common predecessor that's already merged (e.g., both layer onto an
  earlier ticket's foundation).
- Each is sized to ~30-45 min individually.

Examples that worked:
- `auth-01b` (throttle/lockout) + `auth-01c` (password change) — both modify
  `AuthServiceImpl.login` and `AuthController` slightly, but in non-overlapping ways.
- `ai-01b` (cost guard) + `ai-01c` (embeddings) + `ai-01d` (templates) — three siblings
  layered onto `ai-01a`. Each modified `AiServiceImpl` in a different way (`execute` body
  vs new `embed` method vs no change). 6 conflicts on the merge but each was 1-3 min to
  resolve.

## When parallel doesn't make sense

- Tight foundational coupling: the second agent would compile against types the first
  agent is still defining. Run sequentially.
- Both tickets refactor the same method body. Merge becomes hand-coding.
- One ticket is meaningfully larger than the others — it'll dominate the wall-clock time
  anyway, and the others sit waiting on its merge.

## How to set up parallel agents

For a sibling-trio (typical maximum):

1. **Confirm the foundation ticket is merged on main.** All siblings rebase onto it.
2. **Spawn N agents in a single message** with `run_in_background: true`. The Agent tool
   handles parallel scheduling via `isolation: "worktree"` — each gets its own checkout.
3. **Each agent's prompt explicitly names the others' scope walls.** Example:
   ```
   You are running in PARALLEL with two sibling agents:
   - auth-01c (password change) — adds PUT /password endpoint + changePassword method.
     Do NOT modify AuthController or AuthServiceImpl.changePassword.
   - auth-01d (admin endpoints) — adds AdminAuthController. Do NOT modify any /admin/* paths.
   You all share `application.properties` and `AuthSecurityConfig`. Stay in your scope.
   ```
4. **Pause until all agents return**. The notifications come asynchronously; you'll get
   one per agent.

## How to merge

Order matters. Pick the agent that:

- Modifies the **fewest** shared files first. New SPI types, additive helpers, anything
  that doesn't conflict with siblings.
- Has the cleanest commit history (single commit, sane scope).

Merge sequence:

```
1. Pick the simplest sibling. Commit on its own feature branch, push, open PR, wait CI green, merge to main.
2. cd into the next sibling's worktree. Commit. `git rebase origin/main`.
3. Resolve conflicts. (Most are auto-mergeable. Watch for the same method/file modified
   by both — those need hand-merging.)
4. Run `mvn verify` locally — the rebase changes might break the agent's own tests.
   Fix the small breakages in test files (constructor signatures, etc.).
5. Push, open PR, wait CI, merge.
6. Repeat for any remaining siblings.
```

## Common conflict patterns

| Conflict | Fix |
|---|---|
| Both add new fields to `@ConfigurationProperties` record | Combine: include all fields in declaration order, defaults preserved by compact constructor |
| Both add new methods to a service impl class | Both new methods kept in resolution; verify constructor injection covers all new collaborators |
| Both add new entries to an enum | Combine: enum values merged in alphabetical order |
| Both add new exception handlers in `GlobalExceptionHandler` | Both kept in resolution |
| Both add new properties to `application.properties` | Both kept |
| Both add new YAML paths in `openapi.yaml` | Both kept |
| Same line (`testcontainers.bom` version, etc.) | Pick the higher version; test |

In the parallel `ai-01b/c/d` run, total merge resolution was ~30 min for 3 siblings. About
10 min per sibling pair, on average.

## What to budget

- **Agent runtime**: roughly the same as serial. Each agent runs at its own pace.
- **Wall-clock time**: max of the slowest agent's runtime + (N-1) × 10 min for sequential
  merge resolution.
- **Worst-case spike**: if two siblings refactor the same method (which a good ticket
  split avoids), expect 30+ min of hand-merging. If you see this happening, abandon the
  parallel attempt and run sequentially next time.

## Sizing tickets to minimise merge overhead

Lessons from the source project:

- **Each sibling owns its own entity / service / dto stack.** No shared root entity unless
  one ticket explicitly establishes it.
- **Shared cross-cutting files** (`GlobalExceptionHandler`, `AiProperties`, OpenAPI) accept
  additive changes from each sibling. They become merge zones; that's fine if each
  sibling's contribution is a self-contained block.
- **`application.properties` is a merge zone**: each sibling adds its own config block,
  separated by comments. No re-ordering, no consolidation.
- **A constructor with N parameters across siblings is OK**: each sibling's prompt knows
  it'll be expanded; tests using the constructor get updated by the merging human.

## Limits

We've validated up to 3 parallel agents (auth-01b/c stage 2, ai-01b/c/d stage 2). 4+ would
likely be possible if the tickets are very disjoint, but conflicts compound: 4 siblings ×
3 conflicts each = 12 conflicts to resolve. Probably faster to run 4 in two batches of 2.

The hidden cap is **how many distinct mental models you can hold**. Reviewing 3 sibling
agents' reports + resolving 6-9 conflicts is already heavy. 4+ is exhausting.

## When worktree isolation is unavailable

Some Claude Code session configurations report "not in a git repository" and refuse to
create worktrees, even though `.git/` is present in the working directory (the harness's
git detection runs at session start and isn't reactive to later state). When this happens
the Agent tool's `isolation: "worktree"` flag fails outright.

**Fallback pattern**:

1. Parent creates a feature branch off main: `git checkout -b <ticket-id>`.
2. Spawn the agent without `isolation: "worktree"`. The agent works on the live
   working tree, on the branch you just created.
3. Parent verifies, commits, pushes, opens PR, merges.
4. Repeat for the next sibling.

**Cost of falling back**: agents can no longer run in true parallel — they share the
working tree. So an N-sibling round becomes serial (N × ticket-length wall-clock) rather
than parallel (max(ticket-length) wall-clock + merge overhead). For 4-way Wave 2 rounds,
this is ~3-4× slower in wall-clock terms.

**Mitigation**: split parallel rounds into small batches (2 at a time, in 2 sessions) if
you have wall-clock budget. Or accept the serial pace and use the time you save not
context-switching to write tighter ticket specs for the next batch.

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

## Manual-worktree parallel pattern (validated, 3-4× speedup)

If the harness's `isolation: "worktree"` flag fails (typically because the SESSION's primary cwd isn't a git repo even though the project subdirectory IS), you can still get true parallel execution by creating worktrees yourself. This was validated on a 4-module round in the source project: 4 implementation agents ran simultaneously, total wall-clock ~78 min vs the serial equivalent's ~5 hours.

**Setup (parent, before spawning agents):**

```
# Pre-condition: tickets already committed to main; main is green; agents
# are about to be spawned for N sibling tickets.

cd <project-root>
git worktree add ../<project>-wt-moduleA -b feat/moduleA-NN main
git worktree add ../<project>-wt-moduleB -b feat/moduleB-NN main
# ... etc, one per parallel ticket
```

**Agent prompt requirements** (stated explicitly so the agent doesn't drift back to the main checkout):

```
Your worktree is at: <absolute-path-to-worktree>
Your branch: feat/<module>-<NN>

You MUST work in <absolute-path>. Use absolute paths in all Read/Edit/Write
calls. For Bash commands, prefix with `cd "<absolute-path>" ; ...` so mvn
and git operate on YOUR worktree.

Three sibling agents are running in parallel right now in their own
worktrees. Do NOT modify any other module's package.
```

**Spawn all N agents in a single message** with `run_in_background: true`. They run truly in parallel.

**Sequential merge as agents complete (parent):**

```
1. As each agent's completion notification arrives:
   a. cd into its worktree
   b. git add -A && git commit -m "<message>"
   c. git fetch origin main && git rebase origin/main
      (picks up any siblings already merged; per-module file scope means
      conflicts are usually clean appends to entry files like openapi.yaml)
   d. git push -u origin <branch>
   e. gh pr create
   f. gh pr checks <PR> --watch (in background)
2. As CI on each PR goes green:
   a. gh pr merge <PR> --squash --delete-branch
   b. git worktree remove ../<project>-wt-<module>
   c. git branch -D feat/<module>-<NN>
3. After all merged: git checkout main && git pull --ff-only
```

**Skip parent-side local `mvn verify` under parallel.** With N concurrent JVMs + Testcontainers + spotless, Docker daemon and Mockito self-attach (Windows specifically) can produce flakes on tests the agent never touched. CI on isolated GitHub runners is the source of truth. The agent's module-level tests passing in its own report is enough signal to push.

**The worktree itself is the source of truth, not the agent's text reply.** If the parent session pauses and an agent reply message times out (the user comes back hours later, no notification reaches them), the worktree still has the code on disk. Just inspect `git status` in each worktree, push as-is, and let CI judge. This recovered round 4 cleanly when 3 of 4 agent reports were lost to a long session gap.

**Caveats observed at N=4 on Windows:**
- Docker daemon HTTP 500 on Testcontainers startup ~30% of cross-module ITs during the parallel phase
- Mockito InlineByteBuddyMockMaker self-attach failures on Windows JDK 17.0.2 under multi-fork (51 unrelated tests affected; 0 affected on CI Linux)
- JVM "VM terminated without saying goodbye" under memory pressure with 4× simultaneous mvn verify
- Per-agent runtime rose ~2× (61 min vs 23-25 min serial) because each agent burned cycles fighting flakes — but wall-clock still dropped 3-4× overall

**Probably 3 is a better steady-state on Windows; 4 is feasible but flake-noisy.** On Linux, expect closer to theoretical N× speedup.

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

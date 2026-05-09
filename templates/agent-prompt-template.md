# Agent prompt template

The agent's prompt is your single biggest leverage point on per-ticket time. Two failure modes to avoid:

1. **Under-context agent**: prompt is "implement X per ticket" → agent spends 30+ tool calls reading files to find conventions, examples, current state. Burns 5-10 min on round-trips.
2. **Over-context agent**: prompt dumps every doc the project has → agent's context is full before it's done thinking, gets lazy on the actual implementation.

The sweet spot: **front-load the specific snippets the agent would otherwise read**, but stay focused on this ticket's scope.

## Structure

```
You are implementing **<TICKET-ID> — <Title>** for <project description>.

## Read these first, in this order

1. tickets/<module>/<NN>-<title>.md — the ticket. Behavioural spec, file list, edge-case checklist.
2. lld/<module>.md — module design. The relevant sections are §X, §Y, §Z (cite specifically).
3. lld/style-guide.md — project-wide conventions.
4. lld/implementation-playbook.md §Verification model — the verify loop you will run after writing code.

## Shape references — read these exact files

INSTEAD OF `see auth/AuthServiceImpl.java for shape`, INCLUDE the specific snippets the
agent would otherwise have to discover. For example:

> The pattern for a JPA entity with JSONB columns and a `@CreationTimestamp` audit
> column lives at `core/audit/domain/entity/DecisionLog.java`. The relevant pattern:
>
> ```java
> @Entity @Table(name = "decision_log")
> public class DecisionLog {
>   @Id @Column(name = "id", updatable = false, nullable = false)
>   private UUID id;
>
>   @Type(JsonBinaryType.class)
>   @Column(name = "inputs", updatable = false, nullable = false, columnDefinition = "jsonb")
>   private JsonNode inputs;
>
>   @CreationTimestamp
>   @Column(name = "created_at", updatable = false, nullable = false)
>   private Instant createdAt;
>   ...
> }
> ```

This trades prompt tokens for tool-call round-trips. **Token cost is ~50× cheaper than a
file-read round-trip's wall-clock time** (Anthropic prompt caching pins the stable parts
after first use).

## What's already there (do NOT modify)

- Specific file paths the agent should NOT touch
- Warnings about sibling-ticket scope if running parallel

## What you implement

The ticket's "Files this ticket touches" list, restated. Plus 1-2 sentences per non-obvious
file describing what the agent should produce.

## Hard constraints

- Do NOT git commit or git push — the parent does that.
- Stay strictly inside <ticket-ID> scope. Do NOT touch <list of out-of-scope concerns>.
- If running parallel: list sibling tickets and their scope explicitly.
- Don't refactor unrelated code.
- Don't add comments that just restate code.
- <project-specific gotchas — see "Gotchas" section below>

## Quality bar

Project-specific correctness requirements. Examples:

- Test assertions must be specific values, not "method was called" — Pitest catches the rest.
- Security-critical code: list specific invariants (timing parity, generic error messages, etc.).
- Architectural rules: ArchUnit will fail if you violate the cross-module-repo-access rule.

## Verify loop — run AFTER writing the code

```
1. ./mvnw -Dspotless.check.skip=true verify
2. On failures: read errors, fix smallest possible thing, goto 1
3. Cap at 5 iterations; STOP and report if still red after 5
4. ./mvnw spotless:apply
5. ./mvnw spotless:check
```

Do NOT git commit. Do NOT git push. Parent owns commits.

## Output format when done

- Files created (count + key new ones)
- Files modified (with reason for each)
- Final test count after the verify loop converged (e.g., "208 unit + 73 IT all passing")
- Iteration log: how many fix iterations, one-line description of each fix
- Decisions you had to make and want the parent to confirm
- Anything stubbed or skipped

Be deliberate — if you can't make the loop converge in 5 iterations, that's important data.
Stop and report rather than ship red.
```

## Gotchas section — project-specific

Maintain a list of "things every agent on this project has tripped over." Examples from
the source project:

- Postgres `text[]` columns are brittle in Hibernate; use `jsonb List<String>` via
  `@Type(JsonBinaryType.class)` instead.
- `@EntityGraph` over multiple `@OneToMany List<>` collections triggers
  `MultipleBagFetchException` — drop it; lazy loading inside `@Transactional` is fine.
- Use `saveAndFlush` (not `save`) when the controller response payload depends on
  `@Version` increment.
- Markdown loaders that use `\n` in regex break on Windows CRLF — use `\R` (any line
  terminator).
- `docker-java` 3.3.6 + Docker Desktop 29.x → API mismatch. Pin
  `src/test/resources/docker-java.properties` with `api.version=1.44`.
- Multiple `@RestControllerAdvice` beans + an `@ExceptionHandler(Exception.class)`
  catch-all → without explicit `@Order`, the catch-all swallows module-specific
  exceptions and you ship 500s instead of the intended 4xx mappings. Annotate the
  catch-all advice with `@Order(Ordered.LOWEST_PRECEDENCE)` and module-specific
  advices with `@Order(Ordered.HIGHEST_PRECEDENCE)`. Default ordering is unreliable
  across Spring versions; specify explicitly.

Each gotcha is a 30-min tax avoided. Worth keeping in the prompt.

## Token-cost vs round-trip-cost calculation

For Claude Sonnet 4.6 / Haiku 4.5:

- A 200-token snippet costs ~£0.0006 in prompt tokens (cached after first use: ~£0.00006).
- A file-read round-trip costs ~2s of wall-clock latency, which at the user's effective
  hourly rate is worth £0.50+.

So embedding 1KB of shape snippets saves ~30s+ of agent runtime on a typical ticket
where the agent would otherwise read ~15 files to find conventions. Net: **dollars saved
per ticket**, even ignoring the user's time saving from faster ticket turnaround.

The exception: never embed the full LLD or full ticket — those need to be read so the
agent traces requirements back to source. But the snippet-of-an-existing-shape pattern is
nearly always net-positive.

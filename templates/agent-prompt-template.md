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

## When running in a worktree (parallel pattern)

If your prompt directs you to a worktree path (e.g., `C:\path\to\project-wt-foo`), you are running in PARALLEL with sibling agents in their own worktrees. Conventions:

- Use ABSOLUTE paths in every Read/Edit/Write call. The path you were given IS your project root.
- Prefix every Bash command with `cd "<your-worktree-absolute-path>" ; ...` so mvn/git operate on YOUR worktree, not the parent checkout.
- Don't `cd` into a sibling worktree or the main project tree — your worktree is a complete checkout of the same codebase.
- Don't modify other modules' files — siblings are working there.

**Verify loop in a worktree** can flake on cross-module tests under parallel pressure (Testcontainers Docker overload, Mockito self-attach on Windows, JVM memory). If your NEW tests (`-Dtest=YourNewTest -Dit.test=YourNewIT`) pass in isolation but cross-module tests flake, that's an environment issue — note it in your report and let the parent rely on CI. Don't iterate the local loop more than 2-3 times chasing environmental flakes.

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
- **OpenAPI 3.0 `nullable: true` next to `$ref` is silently ignored** by
  swagger-parser. Sibling keywords on a `$ref` are dropped — only the ref's
  target schema matters. For nullable enums, inline `{ type: string, enum: [...],
  nullable: true }` rather than `{ $ref: '#/MyEnum', nullable: true }`. For
  nullable nested objects, inline the entire `type: object` + `properties` block
  with `nullable: true` directly. The `allOf + nullable: true` wrap doesn't help
  either — the inner allOf schema still rejects null.

  **Even if you've defined the named schema for documentation purposes**, don't
  reuse it via `$ref` when the parent field is nullable. INLINE the schema in
  the parent DTO's `properties:` block. The DRY instinct (reuse named schemas)
  is the wrong instinct here — duplicate the structure. This trap has hit
  Round 1, Round 4, AND Round 6 of the source project; it's sticky because
  reusing feels right. Override the instinct.
- **Spring `Page<T>` response bodies** include `pageable` and `sort` properties
  that named page schemas reject unless explicitly `additionalProperties: true`.
  Add it to every `<X>DtoPage` schema you declare.
- **`replaceChildren()` then `saveAndFlush` trips `(parent_id, business_key)`
  unique constraints** when Hibernate flushes inserts before deletes within one
  flush. Compute the change-set BEFORE mutating the aggregate; only mutate +
  flush when changedFields is non-empty. The naive pattern works for collections
  without business-key uniqueness (preference's allergies are list-of-strings)
  but fails as soon as a child has a natural key.
- **Don't trust LLD column widths blindly.** Compute from the format. Recipe's
  LLD spec'd `created_by_actor varchar(32)` but the contract value
  `'user:<uuid>'` is 41 chars. If a column holds a templated string, size from
  the longest possible value, not the LLD's number.
- **HTTP-client adapters (RestClient / WebClient) must live in `..api..` or
  `..config..`** — the project's ArchUnit `springWebStaysInApi` rule forbids
  Spring Web types in `domain.service.internal`. If you need to fetch a URL or
  call an external service, place the adapter in `<module>/config/` (or
  `<module>/api/internal/`). Recipe-01b's `UrlFetcher` was caught by this rule
  in iter 1 and fixed by relocation; bake the convention in upfront.
- **Boolean DTO fields with a default but no explicit `nullable`**: Jackson
  serialises an unset `Boolean` (boxed) field to JSON `null`, which the
  swagger-request-validator rejects against a non-nullable `boolean` schema.
  Either declare the schema `nullable: true` (matches the `default: false`
  intent) OR use a primitive `boolean` field on the DTO so unset stays `false`
  (no null in JSON).
- **YAML inline-flow `{ ... }` strings with internal commas/colons/quotes**:
  in flow style, commas separate map entries — so `description: foo, bar baz`
  inside `{ ... }` is parsed as TWO entries (`description: foo` then
  `bar baz:`). swagger-cli rejects the malformed Response Object as
  "must NOT have additional properties / must have required property '$ref'".
  Always single-quote OpenAPI description strings if they contain commas,
  colons, or quotes — or use multi-line block style for descriptions >3 words.
- **MockMvc `put("...%XX...")` double-encodes URL paths**: MockMvc treats the
  string as a URL template and re-encodes the `%` character, producing
  `%25XX`. Spring's `StrictHttpFirewall` blocks URLs containing `%25` in
  path segments → 400 with no handler matched (Handler = null in mock logs).
  Workaround: for IT tests, use path-variable values that don't require URL
  encoding (`chicken-breast` not `chicken breast`). Move encoding-sensitive
  coverage to unit tests on the normaliser/decoder, not HTTP layer.

## SPI-with-Noop pattern gotcha cluster

If the ticket asks you to define a Service-Provider-Interface (SPI) with a Noop / default fallback that another module will override later, you'll hit a chain of four bugs unless you follow this exact recipe:

```java
// CORRECT: @Bean factory in a @Configuration class with a DISTINCT method name
@Configuration
public class NoopMyServiceConfiguration {     // class name → bean "noopMyServiceConfiguration"

  @Bean
  @ConditionalOnMissingBean(MyService.class)
  MyService defaultMyService() {              // method name → bean "defaultMyService" (DIFFERENT)
    return new NoopMyServiceImpl();
  }

  static class NoopMyServiceImpl implements MyService { ... }
}
```

**Do NOT use**:
- `@Component @ConditionalOnMissingBean` on the class itself — the conditional fires during component-scan, before other beans (test configs, sibling-module configs) register. The Noop registers unconditionally, then a real impl appears later → `NoUniqueBeanDefinitionException`.
- A `@Bean` method named the same (case-insensitively) as the enclosing `@Configuration` class — both register a bean with the same auto-generated name → `BeanDefinitionOverrideException` at startup, ALL ITs fail to load context.

**Test-side bean override**: when a test wants to provide its own implementation via `@TestConfiguration`, add `@Primary` to the test bean. Spring's `@ConditionalOnMissingBean` doesn't always defer to `@TestConfiguration` imports because the imports may register after the conditional has already evaluated.

## `@TransactionalEventListener` + `@Transactional` propagation

If you put `@Transactional` on a `@TransactionalEventListener` method, Spring REQUIRES the propagation to be either `REQUIRES_NEW` or `NOT_SUPPORTED`. The default (`REQUIRED`) and `SUPPORTS` are rejected at context-load with a fail-fast error that blocks EVERY IT across the project — not just the one with the listener.

```
@TransactionalEventListener method must not be annotated with @Transactional
unless when declared as REQUIRES_NEW or NOT_SUPPORTED
```

For an `AFTER_COMMIT` listener that needs to do JPA work (read entities, write back via an SPI, etc.), the publisher's tx has committed and closed by the time the listener runs — so the listener body has no active tx. JPA operations inside it throw `TransactionRequiredException`. Even if the listener eventually calls a `@Transactional` service method (which opens its own tx), the calls BEFORE that point are still outside a tx and fail.

Recipe:

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void onSomeEvent(SomeEvent event) {
  // JPA reads + writes here run in a fresh tx
}
```

## `@MockBean` on a multi-interface `@Service` class

If the real `@Service` class implements MULTIPLE interfaces (common for facade-style services — e.g., `RecipeServiceImpl implements RecipeQueryService, RecipeUpdateService, RecipeSubstitutionRecorder`), `@MockBean private SomeInterface foo` removes the entire real bean AND substitutes a mock that satisfies ONLY that one interface. Spring then fails to wire any other interface the real class provided → `NoSuchBeanDefinitionException` at context load, EVERY test in the IT errors out with "expected at least 1 bean".

Fix: `@MockBean` every interface the real class implements, even if you only stub behavior on one:

```java
// Real class: @Service class RecipeServiceImpl implements
//   RecipeQueryService, RecipeUpdateService, RecipeSubstitutionRecorder { ... }

@MockBean private RecipeQueryService recipeQueryService;       // you wanted to stub this
@MockBean private RecipeUpdateService recipeUpdateService;     // collateral — needs a stub too
@MockBean private RecipeSubstitutionRecorder recipeSubRecorder; // same
```

Quick check before adding `@MockBean`: `grep "implements" src/main/java/.../<TargetClass>.java` to see all interfaces it provides.

## `@Transactional` + intentional 4xx-throwing service methods

If a service writes audit/state/verdict rows then throws an exception that maps to a 4xx (e.g., `BlockedByValidationException` mapped to 422), the default `@Transactional` behavior rolls back EVERYTHING — including the audit rows you just wrote. The response is correct, but the user can't review the verdict because the row doesn't exist.

Fix: `@Transactional(noRollbackFor = YourException.class)` on the service method. Intermediate writes commit; the exception still surfaces. Test by reading back via JdbcTemplate after the 4xx response.

## Don't skip `spotless:apply`

The verify-loop section below says "run `./mvnw spotless:apply` then
`./mvnw spotless:check` on green". This is NOT optional. CI's spotless check
will fail your PR if format violations remain, costing a round-trip
(~6 min CI + parent push). **Run both, every time.** If your final report
says "verify green", the spotless commands MUST have been run; otherwise
the report is misleading.

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

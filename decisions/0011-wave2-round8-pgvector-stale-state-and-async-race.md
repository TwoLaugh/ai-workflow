# Decision 0011 — Wave 2 round 8 (pgvector + Hibernate `StaleObjectStateException`; async-listener test races)

**Date**: 2026-05-12
**Status**: Round 8 complete — 3 modules shipped (household has no 01h; 3-way parallel)
**Source**: TwoLaugh/foodSystem PRs #38–#40

## What landed

| Ticket | PR | CI rounds | Parent fixes |
|---|---|---|---|
| nutrition-01h (IntakeAggregator + WeeklyAggregateDto + DivergenceDetector, DIRECT in-method invocation — sidesteps round-7 listener rule) | [#40](https://github.com/TwoLaugh/foodSystem/pull/40) | 2 (1 infra flake) | 0 |
| provisions-01h (GroceryImportProcessor + idempotency log table on (user_id, source, source_ref UNIQUE)) | [#39](https://github.com/TwoLaugh/foodSystem/pull/39) | 1 | 0 (clean ship) |
| recipe-01h (pgvector vector(1536) + HNSW index + AttributeConverter + async embedding listener) | [#38](https://github.com/TwoLaugh/foodSystem/pull/38) | 4 | **3** (pgvector binding + StaleObjectState + test asserts) |

**Round-8 wall-clock**: ~3h from spawn to all-merged — dominated by the recipe-01h pgvector + JPA fight.

## The big new gotcha: pgvector + Hibernate `StaleObjectStateException`

`recipe_versions.embedding` is `vector(1536)` (pgvector). The async listener fires `AFTER_COMMIT` and calls `RecipeWriteApi.storeEmbedding(versionId, float[], modelId)` to persist the vector + flip `embedding_status` from `'pending'` to `'embedded'`. The original implementation did the obvious thing: `findById → setters → saveAndFlush`. Three back-to-back failures shipped this fix:

### Attempt 1 — plain bind: SQLState 42804

```
ERROR: column "embedding" is of type vector but expression is of type character varying
```

The `AttributeConverter<float[], String>` renders the vector as the pgvector text literal `'[v1,v2,...]'`. Hibernate binds it via `setString`. Postgres refuses the implicit `varchar → vector` cast.

### Attempt 2 — `@JdbcTypeCode(SqlTypes.OTHER)`: 42804 again

The intuition was that `SqlTypes.OTHER` would route the bind through `setObject(idx, value, Types.OTHER)` so Postgres' implicit cast for `unknown → vector` would trigger. In reality Hibernate routed the String as `bytea`:

```
ERROR: column "embedding" is of type vector but expression is of type bytea
```

### Attempt 3 — `@ColumnTransformer(write = "?::vector")`: cast works, but...

This wraps the bound parameter in an explicit SQL `CAST` at write time, keeping the standard `varchar` bind:

```java
@Convert(converter = RecipeEmbeddingConverter.class)
@Column(name = "embedding", columnDefinition = "vector(1536)")
@ColumnTransformer(write = "?::vector")
private float[] embedding;
```

The cast worked. **But the next test failure was a fresh class of bug**:

```
org.hibernate.StaleObjectStateException:
Row was updated or deleted by another transaction
(or unsaved-value mapping was incorrect): RecipeVersion#<uuid>
```

The async listener fires `AFTER_COMMIT` on the create flow. By the time the listener's `findById` loads a fresh `RecipeVersion` (with cascaded `ingredients` + `methodSteps` + `metadata` + `tags`), Hibernate's full-entity dirty-check sees the entity state diverge from what's in the row (the persistence context from the create flow is gone, but its bookkeeping leaves the row at a state the listener's load doesn't agree with). `saveAndFlush` then fires UPDATE…WHERE id=… AND <whatever-version-state> and gets 0 rows back → throws `StaleObjectStateException`. Notably, `RecipeVersion` has **no** `@Version` field; the stale-state detection isn't optimistic locking, it's Hibernate's append-only entity heuristic.

### Attempt 4 — native UPDATE: green

Two `@Modifying @Query(nativeQuery=true)` methods on `RecipeVersionRepository`:

```java
@Modifying
@Query(value =
    "UPDATE recipe_versions SET embedding = CAST(:embedding AS vector),"
  + " embedding_model_id = :modelId, embedded_at = :embeddedAt,"
  + " embedding_status = 'embedded' WHERE id = :id",
    nativeQuery = true)
int updateEmbedding(...);

@Modifying
@Query(value = "UPDATE recipe_versions SET embedding_status = 'failed' WHERE id = :id",
    nativeQuery = true)
int markEmbeddingFailed(@Param("id") UUID id);
```

`RecipeServiceImpl.storeEmbedding` / `.markEmbeddingFailed` now call the repo methods directly, side-stepping Hibernate's full-entity dirty-check entirely. The native SQL `CAST(:embedding AS vector)` makes the `@ColumnTransformer` on the entity field redundant for the write path — but the converter still serves reads.

### Why the entity-mutation tests broke

`RecipeWriteApiTest` (a unit test with `@Mock versionRepository`) used to stub `findById` + `saveAndFlush` and assert the in-memory entity was mutated. After the refactor:
- The code never mutates the entity, only calls the native UPDATE.
- Stubs needed to return rowcount `1` (success) or `0` (not-found) for `updateEmbedding` / `markEmbeddingFailed`.
- Entity-mutation assertions had to be dropped; replaced with event-content assertions (reason + recipeId + versionId).

## The other class of bug: async-listener test races

`RecipesFlowIT.post_returns201` reads `embedding_status` from the DB after POST returns and asserted `"pending"`. With the listener actually wired up (and the stub `AiService` returning instantly), the async listener flips `'pending' → 'embedded'` before the assertion reads it. The synchronous JSON response body already asserts `'pending'` on a different line; the DB re-read was redundant + racy. Fix: loosen DB assertion to `isIn("pending", "embedded", "failed")` — any terminal state is valid post-commit.

## What worked

- **Local Docker validation before each push**: the StaleObjectState only manifests under a real Postgres + real Hibernate session. Without Docker (1st attempt), I pushed two broken commits to CI before Docker came back up. With Docker (3rd+ attempts), each fix was validated in <3 min locally before pushing.
- **Surefire `.txt` reports** are gold for diagnosing context-load failures: the truncated `mvnw -q` output buries the root cause, the per-class `.txt` shows the full `Caused by` chain.
- **3-way parallel**: nutrition and provisions both shipped in 1 round each with zero parent fixes, hiding completely behind the recipe-01h slog.

## What didn't

- Wasted 2 commits on the pgvector binding before landing `@ColumnTransformer`. The `@JdbcTypeCode(OTHER)` blind alley was based on a stale Stack Overflow pattern from pgvector-jdbc 0.0.x; the current `pgvector-java` docs are explicit about `@ColumnTransformer(write = "?::vector")` being the canonical approach but I didn't read them first.
- Stale surefire reports across runs caused false alarms: a `*Recipe*` test sweep across 9 ITs had 5 errors that all turned out to be "Could not find a valid Docker environment" left over from an earlier run when Docker was down. Lesson: `target/surefire-reports/*.txt` are NOT auto-cleared between runs — check the timestamp before trusting them.

## New rules baked into the prompt template

1. **pgvector entity columns**: combine `@AttributeConverter<float[], String>` (renders `'[v1,...]'`) with `@ColumnTransformer(write = "?::vector")` on the field. Do NOT try `@JdbcTypeCode(OTHER)`.

2. **Async `AFTER_COMMIT` listeners that need to mutate an entity from the publisher's flow**: use a native `@Modifying @Query` UPDATE on the repository, NOT a `findById → setters → saveAndFlush` round-trip. The latter races with Hibernate's lingering dirty-check state from the publisher's persistence context and throws `StaleObjectStateException` even on entities with no `@Version` field.

3. **Tests that assert post-state of fields the async listener mutates**: either loosen the assertion to a set of valid terminal states, or block the listener via a `@MockBean` / `@TestConfiguration` that captures the event without invoking the body. Asserting the initial value after the response has returned is inherently racy.

4. **Surefire report hygiene**: when interpreting `target/surefire-reports/*.txt`, check timestamps. Reports persist across runs — a failure in the report doesn't mean a failure in the most recent run unless the timestamp matches.

## Round 8 summary

| Metric | Value |
|---|---|
| Modules | 3 (household has no 01h) |
| PRs | 3 (#38, #39, #40) |
| Total CI rounds | 7 (4 recipe, 2 nutrition incl. infra flake, 1 provisions) |
| Parent fixes | 3 (all on recipe-01h) |
| Wall-clock to all-merged | ~3h |
| New rules added to prompt template | 4 |

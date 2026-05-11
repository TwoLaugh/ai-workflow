# Decision 0008 — Wave 2 round 5 (SPI-pattern friction; recovery from session restart)

**Date**: 2026-05-11
**Status**: Round 5 complete — 4 implementation tickets shipped
**Source**: TwoLaugh/foodSystem PRs #27–#30

## What landed

| Ticket | PR | Files | CI rounds | Real bugs caught |
|---|---|---|---|---|
| household-01e (merge service + SoftPreferencesReader SPI) | [#28](https://github.com/TwoLaugh/foodSystem/pull/28) | 27 | 2 | Spring `@TestConfiguration` bean vs `@ConditionalOnMissingBean` ordering — fixed with `@Primary` on test bean |
| nutrition-01e (HealthDirective queue + safety gate) | [#29](https://github.com/TwoLaugh/foodSystem/pull/29) | 42 | 4 | 3 issues stacked: `ResponseStatusException` in `domain.service.internal` (ArchUnit `springWebStaysInApi`); `@Component @ConditionalOnMissingBean` doesn't compose; `@Configuration` class + same-named `@Bean` method → `BeanDefinitionOverrideException` |
| provisions-01e (waste log + WasteValidator) | [#27](https://github.com/TwoLaugh/foodSystem/pull/27) | 33 | 1 | None — agent caught ArchUnit violation locally in iter 2, shipped clean |
| recipe-01e (RecipeSubstitution + state machine + overlay) | [#30](https://github.com/TwoLaugh/foodSystem/pull/30) | 42 | 1 | None — green first try |

**Round-5 wall-clock**: ~96 min for the agents to implement + a long session restart in the middle. Effective parent-time on the recovery side: ~40 min across 4 PRs (3 with parent fixes).

## The session-restart recovery worked

Halfway through round 5, the parent session restarted, killing the 3 remaining background agents (provisions had already merged). All 3 worktrees retained their work on disk (27, 42, 42 changed files). The recovery pattern from round 4's playbook applied cleanly:

1. Spotless apply per worktree
2. Commit
3. Rebase onto current main
4. Push, open PR
5. Let CI judge

3 of 3 went green eventually — though each needed at least one parent fix because the agents never got the chance to do their own verify loop.

**Lesson confirmed**: "the worktree is the source of truth, not the agent's text reply" — this is now the second time the pattern recovered cleanly.

## The new gotcha cluster: SPI pattern under Spring conditional ordering

Round 5 introduced the SPI-with-Noop pattern in two places (household + nutrition). Both hit the same chain of Spring bean-conditional bugs. **This is the new gotcha class** to add to ai-workflow templates:

### Bug 1: `@Component @ConditionalOnMissingBean` doesn't compose at component-scan time

The agent's natural first instinct is:
```java
@Component
@ConditionalOnMissingBean(MyInterface.class)
public class NoopImpl implements MyInterface { ... }
```

This DOES NOT WORK reliably. The `@ConditionalOnMissingBean` condition fires during component scan, BEFORE other beans (test configs, other module configs) have been processed. The Noop registers itself unconditionally, then a later bean (e.g. via `@TestConfiguration`) registers a second one → `NoUniqueBeanDefinitionException`.

**Fix**: use `@Bean @ConditionalOnMissingBean` factory method inside a `@Configuration` class. `@Bean` conditions evaluate after bean definition gathering completes, so they correctly defer to real implementations.

### Bug 2: `@Configuration class Foo { @Bean Foo foo() }` → name collision

When you put `@Bean methodName()` inside `@Configuration class ClassName`, BOTH register beans. By default, both get the lowercased class/method name. If `ClassName` and `methodName` lowercase to the same string → `BeanDefinitionOverrideException` at startup.

**Fix**: rename the method (or class) so the auto-generated bean names don't collide. Convention: name the method `default<InterfaceName>` and the class `<InterfaceName>NoopConfiguration` or similar.

### Bug 3: `@TestConfiguration` bean + production `@ConditionalOnMissingBean` → both register

Even with the `@Bean @ConditionalOnMissingBean` factory pattern, a `@TestConfiguration` bean imported via `@Import` may register too late to be visible to the conditional. Result: both Noop and Fake exist → ambiguous wiring at injection site.

**Fix**: add `@Primary` to the test-time bean. Spring picks `@Primary` over conditional fallbacks unambiguously.

### Bug 4: `@Transactional` rolls back when service throws to surface a 4xx

If the service writes audit/verdict rows then throws `SomethingException` (mapped to 4xx by the handler), the surrounding `@Transactional` rolls back the writes by default. The 4xx returns correctly, but the rows are gone.

**Fix**: `@Transactional(noRollbackFor = MyException.class)` on the method when you want intermediate writes to commit even though the request fails.

## Per-ticket runtime + parent-fix cost

- **provisions-01e**: 0 parent fixes. Agent caught ArchUnit violation in iter 2 locally. Clean ship.
- **recipe-01e**: 0 parent fixes. Green first try after restart-recovery push.
- **household-01e**: 1 parent fix (8 lines of test config — add `@Primary`). 1 CI round.
- **nutrition-01e**: 3 stacked parent fixes (ArchUnit, Noop refactor, bean-name rename). 3 CI rounds.

**Cost asymmetry**: nutrition's HealthDirective ticket introduced TWO SPI patterns simultaneously (`DirectiveApplyTarget` + the transactional safety-gate verdict write). Each pattern triggered its own bug class. The other 3 tickets had simpler shapes.

**Lesson**: when a ticket introduces TWO unfamiliar patterns at once (here: SPI + transactional rollback semantics), expect 2-3× the parent-fix budget vs a single-pattern ticket. Worth flagging in the ticket-writer prompt.

## Round-on-round comparison

| Round | Style | Wall-clock | Agent verify avg | Parent fixes avg | Real bugs caught |
|---|---|---|---|---|---|
| 1 | serial | ~4 h | 2 | 1.25 | 5 |
| 2 | serial | ~5.5 h | 1.25 | 0 | 0 |
| 3 | parallel | ~1.3 h | 2-5 (contention) | 0 | 0 |
| 4 | parallel | ~3-4 h (session gap) | uncaptured | 0.75 | 3 |
| 5 | parallel + restart | ~1.5 h + restart recovery | uncaptured (mostly) | 1.0 | 4 |

Parent-fix count keeps drifting up as the ticket complexity rises. That's expected — the gotcha list grows linearly with how-many-Spring-features-we-use, but the time-per-fix is shrinking (the parent now recognizes the bug classes faster).

## Implications for ai-workflow templates

1. **`templates/agent-prompt-template.md`** — add the SPI bean-pattern gotcha cluster (4 sub-bugs above). When a ticket spec uses the words "SPI", "Noop", or `@ConditionalOnMissingBean`, the agent must use the `@Configuration` + `@Bean` factory pattern AND name the bean method distinctly from the class.
2. **`templates/agent-prompt-template.md`** — add: when a service writes audit/state rows then throws a 4xx exception, mark `@Transactional(noRollbackFor = ...)` so intermediate writes survive. Otherwise the request fails AND the writes get rolled back.
3. **`templates/ticket-template.md`** — flag that tickets introducing TWO new patterns at once (SPI + transactional, async + retry, etc.) need either a pre-split or a doubled parent-fix budget.

## What's next

Per the round-1+2+3+4+5 deferral maps, remaining Wave 2 sub-tickets:

- household: 01f slot-config planner-friendly view (blocked on planner module — defer to Wave 3?)
- nutrition: 01f calc service, 01g floor-gate, 01h aggregation
- provisions: 01f planner bundle, 01g cook events, 01h grocery import, 01i staples, 01j batch-cook, 01k expiry, 01l feedback, 01m pantry gate
- recipe: 01f write API, 01g promotion/archive, 01h embeddings, 01i search, 01j helpers, 01k AI tag inference

~14 sub-tickets left in Wave 2 alone. At the round-5 pace (~80 min per round of 4), that's ~4 hours of orchestration to finish backend Wave 2. Then Wave 3 (planner, discovery, feedback module).

The pattern is mature. Recipe pacing is on track.

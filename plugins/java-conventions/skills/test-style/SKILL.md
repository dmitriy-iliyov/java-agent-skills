---
name: test-style
description: The testing style for a Java project - how tests are named, what each of the three levels (UT/IT/CT) is responsible for, what may be mocked and what may not, and what has to be asserted at each level. Load it before writing or rewriting a test, and whenever judging whether an existing test still follows the house style. The procedure for bringing a module's tests back to green after a refactor is the separate `test-refactor` skill.
---

# Test style

House rules for `src/test`. A test class that breaks one of them is either wrong, or a reason to change the
rule here deliberately.

Code, javadoc and message text are written in English.

## The three levels

| Level | File suffix | `@DisplayName` prefix | What is alive | What is stubbed |
|---|---|---|---|---|
| Unit | `*UnitTest` | `UT` | one class | all of its collaborators |
| Integration | `*IntegrationTest` | `IT` | the Spring context, or a real Postgres/Redis in a container | everything not under test |
| Component | `*ComponentTest` | `CT` | the whole module | only other modules |

**UT** is about the contract of a single class. No Spring context; collaborators are Mockito mocks.

**IT** is for what a UT cannot reach, and it comes in two flavours that share the suffix and little else:

- **IT of an autoconfiguration** - the composition of the Spring context, run on `ApplicationContextRunner`
  or `WebApplicationContextRunner`. Nothing external is alive; the beans this particular test does not
  assert on are mocks.
- **IT against a real external system** - the classical integration test, and the one a module that owns a
  store or a client is made of. Testcontainers brings up a real Postgres or a real Redis, and the test
  asserts that the implementation delivers the guarantees its interface promises - against the real engine
  rather than against a mock of it, because those guarantees (atomicity, a conditional write, what a `NULL`
  column reads back as) are exactly what a mocked driver cannot have. A Spring context is not needed here:
  the class under test is constructed by hand over a `JdbcClient` or a `RedisTemplate` pointed at the
  container.

Which flavour a file is has to be readable from its name: `<Module>AutoConfigurationIntegrationTest` is the
first, `PostgreSqlOrderRepositoryIntegrationTest` the second.

**CT** is about the module actually working as a whole. The key rule: **components of your own module are
never mocked.** Mocking one means asserting your own guess about how it behaves instead of the behaviour
itself - which is exactly how the wiring bugs a CT exists to catch get through. Stub the neighbouring
modules only.

## Naming

Method: `method_whenCondition_shouldResult` - lower camelCase, three parts joined by `_`.

`@DisplayName`: `"<level> <method>() when <condition> should <result>"` - normally the same sentence as the
method name, spelled out:

```java
@Test
@DisplayName("UT match() when several patterns match should take the most specific one")
void match_whenSeveralPatternsMatch_shouldTakeMostSpecificOne() { }
```

The display name may say more than the method name when the case needs opening up - a condition an
identifier cannot carry, the reason the outcome is what it is. What it may not be is long: it is read in a
run log, one line among hundreds, so the budget is another clause, not another sentence. A name that starts
wanting a sentence usually means the test is holding two cases.

For a constructor, and for a whole scenario in a CT, the first part is not a method but what happens:

```java
void constructor_whenPrefixIsNull_shouldThrowNullPointerException()
void request_whenSameOrderIsPostedTwice_shouldBeAnsweredFromCacheWithoutReachingHandler()  // CT
void context_whenExistingClockBean_shouldNotRegisterDefaultClock()                         // IT of an autoconfig
```

The result is described as **observable behaviour, not as implementation**:
`shouldReturnNullWithoutResolvingAnything`, not `shouldNotCallResolver`.

Helper classes living in the test source root do **not** carry the `Test` suffix (`TestOrder`, `TestClock`) -
otherwise surefire tries to run them as tests.

## The shape of a test

```java
@Test
@DisplayName("UT getRoutes() after afterSingletonsInstantiated() should return unmodifiable set")
void getRoutes_afterAfterSingletonsInstantiated_shouldReturnUnmodifiableSet() {
    // given
    givenHandlerMethods(annotatedEndpoint("/payments"));
    DefaultRouteRegistry tested = registry("");

    // when
    tested.afterSingletonsInstantiated();

    // then
    assertThatThrownBy(() -> tested.getRoutes().add("/orders"))
            .isInstanceOf(UnsupportedOperationException.class);
}
```

- The `// given` / `// when` / `// then` sections are labelled explicitly. When "when" and "then" cannot be
  separated, write `// when / then`.
- The object under test is called `tested` - a field with `@InjectMocks`, or a local variable when the
  constructor depends on the case.
- One test, one scenario. Several `assertThat` calls are fine as long as they describe the same outcome.
- Data preparation moves into private helpers **at the bottom of the class**, after the tests. The body of a
  test keeps only what makes this case different from the one next to it.
- Beyond those three labels, comments and javadoc are kept to a minimum. One is written only where the
  reader would otherwise not follow what is happening - a setup that is not obvious, a value that means
  something its type does not show, a line at the top of the class saying what the whole file holds down.
  A comment that restates the code, and javadoc on a test method, are noise: the display name has already
  said what the case is.

Assertions are **AssertJ**: `assertThat`, `assertThatThrownBy` from `org.assertj.core.api.Assertions`. The
JUnit `assertEquals`/`assertTrue` and the import from `AssertionsForClassTypes` still found in a couple of old
files are legacy and must not appear in new tests.

`@Nested` earns its place when one function has several independent groups of cases - the modes of a
normalizer, say, each with its own set. For a flat set of cases it is noise.

## Mocks and stubs

Mockito is for **collaborators that only answer** in this test: `@Mock` on the field, `@InjectMocks` on
`tested`, `@ExtendWith(MockitoExtension.class)` on the class. Strict stubs are on by default - a superfluous
`when(...)` fails the test, and that is correct.

A hand-written stub (`Recording*`, `Test*`, `Probe*` - a static nested class) is for when **state or
behaviour** is needed: a call counter, an in-memory store, "invoke the caller's callback". A mock with a
`thenAnswer` that contains logic always loses to such a class on readability. Name it for what it does:
`RecordingProcessor`, `RecordingEventChannel`, `InMemoryOrderRepository`.

Value objects (value types, metadata) are not mocked but built: `TestOrder.builder()` with sane
defaults, overriding one or two fields per case. Such a builder lives in its own module's test package.

"Nothing went here" is asserted with `verifyNoInteractions(...)` / `verify(x, never())`, not by the silence of
the test.

## What each level has to assert

**UT of a class:**
- every `Objects.requireNonNull` in the constructor - one test per argument, asserting the message text;
- every branch of every public method, including "found nothing" (`null`, empty collection) and the handling
  of a collaborator's exception;
- boundary inputs: `null`, empty string, blanks, empty collection;
- what the method does **not** do: did not touch the resolver, did not reach into the cache.

**IT of an autoconfiguration** (`ApplicationContextRunner` / `WebApplicationContextRunner`):
- the full set of beans when every required dependency is present - both by interface and by concrete class;
- `@ConditionalOnMissingBean`: the user's own bean displaces the default one (`hasSingleBean(Iface)` +
  `doesNotHaveBean(Default.class)`) - one test per `@Bean`;
- a missing required bean brings the context down (`assertThat(context).hasFailed()`);
- the conditions themselves: `@ConditionalOnWebApplication` is checked by a run on a non-web
  `ApplicationContextRunner`;
- registration settings, not merely the existence of the bean: filter order, url patterns, path prefix.

**IT against a real external system** (Testcontainers):
- every method of the contract against the real engine, in both outcomes - the condition met and the
  condition missed;
- the negative half of a failed conditional write: not only that it threw, but that the stored row was left
  as it was;
- the absent case, answered the way the contract says: a missing key gives an empty `Optional`, not a
  `null` and not an exception;
- nullable columns in both directions - written as `NULL` on the write that leaves them empty, read back as
  `null` without blowing up;
- the invariants the contract states in prose: which write puts an expiry on a row, which one clears an
  inherited one, which one must not touch a column at all;
- concurrency where the contract promises it - a claim that has to be held until the caller's transaction
  commits is asserted on two connections, because on one it cannot fail.

The container is shared by the class (`@Container static`), and the data is reset per test in `@BeforeEach`:
what is cleaned between cases is the state, not the engine.

**CT:** the scenarios that cannot be assembled out of UTs - the whole request path, a repeat under the same
key, a bypass on a non-idempotent endpoint, a missing header, an error response.

## Tests and bugs that were found

Three rules. The first two look like opposites and are not: one is about a test that asserts the defect, the
other about a test that asserts the contract.

**A green test does not record a bug.** If an assertion passes only because the behaviour is broken, that
test does not enter the repository - it writes the defect down as if it were the contract, and the next
reader has nothing left to tell them apart by. Such a case is inverted, not deleted: assert what the
contract says, and let it fail.

**A bug must still have its test, and that test is red.** This matters more than the suite being green: a
defect nobody wrote a test for is a defect that gets lost. The case is written against the contract the
moment the bug is found, fails against today's code, and goes green with the fix without being touched
again. It is not disabled and not commented out - a `@Disabled` test reports nothing, which is the whole
problem it was meant to solve.

**The fix is not the test author's to make.** Production code is changed by the operator. Finding a bug
means: write the red test, write the finding into the project's open-items document
(a plan, a spec, a backlog - whatever this project keeps open work in) as its own item, with the numbers of
the run and a mark that it is reproduced, say it out loud - and stop there. A red suite whose red is
accounted for by a recorded item is a working state; a production file quietly patched to make it green is
not.

Reproducing is done with a temporary probe class next to the tests (it prints statuses and counters to
stdout). Once the fact is written down there, **the probe class is deleted** - what stays is the red
test.

## Stability

- No `Thread.sleep` and no dependency on real time: time arrives through `Clock` (`TestClock`).
- Identifiers that are compared later are constants (`UUID.fromString("aaaaaaaa-...")`), not
  `UUID.randomUUID()`; a random UUID belongs exactly where "some other key" is meant.
- Global state is cleaned in `@AfterEach` - `RequestContextHolder.resetRequestAttributes()`, for instance.
- Test order carries no meaning: every test prepares its own state.

## Formatting carried over from production code

Four-space indent, lines up to ~120 characters, explicit local types (no `var`), no Lombok, `"...%s"
.formatted(...)` rather than `String.format`, `byte [] bytes` with the space, enum comparison with the
constant on the left. Multi-line parameter lists align under the opening parenthesis; call arguments that do
not fit wrap at an indent of 8, one per line when there are more than two.

---
name: test-style
description: Load it before creating or editing any file under `src/test` - a new test class, one test method added to an existing class, a regression test written as part of a bug fix, a test changed to follow a production change - and whenever judging whether an existing test follows the house style. Holds the testing style for a Java project - how tests are named (`*UnitTest`, `*MultiThreadTest`, `*IntegrationTest`, `*ComponentTest`, `*InfraTest`, `*EndToEndTest`), what each of the six levels (UT/MT/IT/CT/INF/E2E) is responsible for, what may be mocked and what may not, what has to be asserted at each level, and what to do with a bug a test has found. Not for bringing a module's tests back to green after a refactor - that is the separate `test-refactor` skill.
---

# Test style

House rules for `src/test`. A test class that breaks one of them is either wrong, or a reason to change the
rule here deliberately. Code, javadoc and message text are written in English.

**Every question for the operator is asked in one batch, before the first test is written**, so the work is not
interrupted halfway. First read the sources (see where an expectation comes from) and the code under test, and
collect what this skill leaves to the operator - which features of a module are submodules with a CT suite of their
own, the cases whose sources disagree or are silent, a level that does not obviously apply, a project convention that
differs from this skill - each with the candidates and what each source says. A question that comes up only while
writing means a source was read too late; it is still asked, but at the end with the report, and only its case waits
for it. A bug found while writing needs no question: see tests and bugs that were found.

## The six levels

| Level | File suffix | `@DisplayName` prefix | What is alive | What is stubbed |
|---|---|---|---|---|
| Unit | `*UnitTest` | `UT` | one class | all of its collaborators |
| MultiThread | `*MultiThreadTest` | `MT` | one class driven from several threads; a real engine when the guarantee lives there, the context of a module when the race goes through it | nothing the race goes through |
| Integration | `*IntegrationTest` | `IT` | the Spring context or a slice of it, or a real Postgres/Redis in a container | everything not under test |
| Component | `*ComponentTest` | `CT` | the whole module minus its web layer, over a real database and cache in containers | the neighbouring modules and the services outside |
| Infrastructure | `*InfraTest` | `INF` | one mechanism of a starter with one chosen set of plugins, in a real Boot application | nothing of the library; the user's side is test code |
| End-to-end | `*EndToEndTest` | `E2E` | the whole application, started as in production | only what lies outside the process (SMTP, blob storage) |

**UT** is about the contract of a single class. No Spring context; collaborators are Mockito mocks.

**MT** is about a guarantee that exists only under concurrency: a row lock held until commit, `SKIP LOCKED` handing
two pollers disjoint batches, a compare-and-set won by exactly one of ten threads, a counter that loses no update.
One thread cannot break such a guarantee, so a test on one thread proves nothing about it - hence a level of its own,
not a case in a UT or an IT. The guarantee decides between two shapes:

- **A staged interleaving**, for a guarantee about *order* ("the second poller is not blocked while the first holds
  its batch", "a reader does not see the row before the writer commits"). `CountDownLatch`es force exactly the order
  the contract talks about: the first party takes the lock and holds its transaction open, the second acts
  meanwhile, then the first is released. The run is deterministic - one run is enough.
- **A contention burst**, for an *invariant under every interleaving* (exactly one winner, every item moved exactly
  once, no lost update). N threads parked behind one start gate (`CountDownLatch(1)` or `CyclicBarrier(n)`) are
  released together, so they really meet at the contended point. The interleaving is not controlled, so the burst is
  repeated (`@RepeatedTest`, a few dozen runs) - a race seen one run in twenty is a race all the same. What makes it
  a test and not a hope is the assertion: it states the invariant no interleaving may break, never a result that
  depends on which thread came first.

The mechanics, for both:

- **Threads come from a dedicated pool of the size the test needs**, in try-with-resources (`ExecutorService` is
  `AutoCloseable` since Java 19). `close()` waits for every task and, when interrupted, calls `shutdownNow()` and
  keeps waiting - a worker an interrupt does not reach (a JDBC call waiting on a lock) hangs the build all the same.
  Two things stop it: `@Timeout(value = ..., threadMode = SEPARATE_THREAD)` on the test fails it without waiting for
  the stuck thread, but only reports the hang - the thread lives until the forked JVM exits; every wait bounded on
  the engine's side (`lock_timeout`, `Statement.setQueryTimeout`) is what frees it.
- **Every worker reaches the gate before it opens**: the test awaits a `ready` latch each worker counts down,
  otherwise the late ones start after the race is over.
- **Never `CompletableFuture.runAsync` without an executor.** The common pool's parallelism is cores minus one: on a
  four-core CI runner ten parties blocking on a lock run three at a time, and the race never happens. Below a
  parallelism of two it starts a thread per task instead, so the test also behaves differently per machine.
- **Against a real engine, each party has its own connection and its own transaction**, and the pool has at least as
  many connections as there are threads - otherwise they queue for a connection and arrive one by one.
- **Every wait is bounded and its result asserted**: `assertThat(latch.await(15, SECONDS)).isTrue()`,
  `future.get(15, SECONDS)`. An ignored `await(timeout)` or `awaitTermination` result lets the assertions run while
  the workers are still going.
- **A failure in a worker reaches the test.** An exception inside `submit(...)` lands in a `Future` nobody reads; an
  assertion failing off the main thread is lost. Workers return through a `Future` the test `get()`s (which
  rethrows), or store the result and any `Throwable` in an `AtomicReference` the main thread asserts. A worker never
  swallows an exception (`catch (Throwable e) { }`, `.exceptionally(ex -> null)`) unless it is the expected outcome,
  and then it is counted.
- **Shared results go into concurrent types** - `AtomicInteger`, `ConcurrentLinkedQueue`,
  `ConcurrentHashMap.newKeySet()` - not an `int[]` or a list guarded by hand.
- **A party that has to hold a lock waits on a latch, not a sleep**; an expiry comes from `Clock` or from an expired
  row written in the setup, not from waiting it out.
- **Both sides of the guarantee are asserted**: every party got what it should (the second poller was not starved,
  the winner is recorded as the holder), and the parties did not overlap (disjoint batches, never two holders - an
  `AtomicInteger` raised on entry and lowered on exit, whose maximum must stay 1). "At least one succeeded" asserts
  nothing about a race. Every assertion carries `.as(...)` and worker threads carry names, since a red run of a race
  has to explain itself.
- **Every implementation is run**: a class per database dialect, since locking is exactly where Postgres, MySQL and
  Oracle differ - in a library with a module per dialect, each class lives in its dialect's module, nothing chosen at
  run time the way an INF cell is; a parameter per variant of an in-memory structure (a locking and a lock-free one),
  since they keep the same invariant.

**An MT that needs the context of a module or submodule** - a race through the facades over the real database, a
version conflict staged between two transactions - keeps the MT suffix and prefix but lives at the CT level: it
extends that module's CT base class, runs in its context and containers, and enters through the bean a CT enters
through. The mechanics hold unchanged.

**Not an MT**: a load test (a thousand threads, timings, "survives the load") asserts no invariant a smaller burst
would not, and is slow and timing-dependent - if kept, it is a `@Tag("stress")` test (see how tests run). A race in
the memory model of a lock-free structure (visibility, reordering) is beyond what JUnit finds reliably; that is what
OpenJDK's jcstress is for.

**IT** is for what a UT cannot reach. Three flavours share the suffix and little else, and the file name tells which:

- **of an autoconfiguration** (`<Module>AutoConfigurationIntegrationTest`) - the composition of the Spring context, on
  `ApplicationContextRunner` or `WebApplicationContextRunner`. Nothing external is alive; the beans this test does not
  assert on are mocks.
- **against a real external system** (`PostgreSqlOrderRepositoryIntegrationTest`) - the classical one, which a module
  owning a store or a client is made of. Testcontainers brings up a real Postgres or Redis, and the test asserts the
  guarantees the interface promises - atomicity, a conditional write, what a `NULL` column reads back as - which a
  mocked driver cannot have. No Spring context: the class under test is built by hand over a `JdbcClient` or a
  `RedisTemplate` pointed at the container.
- **of a controller** (`AppointmentControllerIntegrationTest`) - one controller's web contract in a
  `@WebMvcTest(XController.class)` slice, its service a `@MockitoBean`: binding, validation, statuses, error bodies,
  access rules. What the controller must look like and answer is for the `spring-conventions` skills
  (`controller-style`, `controller-advice-style`); this skill only places the test.

**Containers and the Spring context.** Spring caches a context and hands it to the next class that asks for the same
one, while a `@Container` is stopped by the JUnit extension when its class ends - the cached context still points at
its port, and the next class fails. So wherever a context outlives a class (CT, INF, E2E), containers are singletons,
one per engine per JVM: started in a static block of the base class or a holder class, wired through
`@DynamicPropertySource`, never `@Container`. `@Container static` belongs to the IT without a context, whose
container lives exactly as long as its class. A machine without Docker: see how tests run.

**CT** is about the module working as a whole. **Components of your own module are never mocked**: mocking one
asserts your guess about its behaviour instead of the behaviour - exactly how the wiring bugs a CT exists for get
through. Only what is not the module's is replaced: **the neighbouring modules** and **the services outside the
application** (a blob storage, a mail or SMS gateway). The module's services, repositories, mappers, caches and
scripts are real, over a real database and cache.

- **Only a module whose service layer has layers gets a CT** - at least two: a `*Facade` or `*Orchestrator`
  coordinating services that use the repositories and the neighbours. Where services call repositories directly with
  nothing above them, the services' UTs and the adapters' ITs cover what a CT would. "Module" is literal - a module
  of the application, or a Maven module of a library.
- **The entry is the module's public service API - the very bean the controller gets.** Where facades are wrapped in
  decorators (a cache, a retry, a picture upload), that is the outermost decorator, injected by name with
  `@Qualifier`; a bean taken by its interface, or the inner `*Impl`, skips the decorators and the CT stays green on a
  path no user takes. The order of the chain is asserted once, where it is assembled: by a CT of its own that unwraps
  it decorator by decorator when the module's configuration builds it, by the IT of the autoconfiguration when a
  starter does. The controller is not part of a CT: its web contract is its IT, the whole HTTP path an E2E.
- **One module, one CT suite** - its `*ComponentTest` classes, one per group of use cases, all on the module's base
  class. A module too big for one suite (several features that would each make a module) gets a suite per submodule
  plus one of the whole module. Which features are submodules the operator decides up front, given the candidates -
  the packages with services, facades or orchestrators of their own. The whole-module suite repeats nothing the
  submodule suites cover: it asserts the hand-overs between submodules and the invariants seen only at module level
  (access rights, data consistent across submodules) - a small E2E inside the module.
- **The database and the cache are real**, singleton Testcontainers: a unique constraint, a rollback, an optimistic
  lock, a Redis script are what a repository fake would not have. One module context without its web layer and one
  set of containers are shared by every CT class through the base class; tables are emptied and the cache flushed
  before each test.
- **A neighbouring module** is a Mockito mock when it only answers, a hand-written fake (`Recording*`, `InMemory*`)
  when the scenario needs its state or a record of calls; **a service outside the application** is an `InMemory*`
  fake. Fakes implement the neighbour's contract interface, are declared once in the module's CT configuration next to
  the base class, and are cleared before each test. They are not checked against the real neighbour: a drifted fake
  is caught by the E2E of a flow, which runs both modules for real.
- **What the module stored is read back through its own API** (the facade or service that answers for it), not out
  of the tables. **What left it is asserted on the fakes** - what the `Recording*` neighbours received, what the
  `InMemory*` services hold - and on the application events it published (`@RecordApplicationEvents`).

**INF** exists **only for a starter or a library** with interchangeable plugins (database dialects, brokers) used
from several kinds of user code (JDBC and JPA, an annotation and an explicit call). It takes one mechanism (publishing
through an outbox, idempotent consumption) and checks it the way a user gets it, across that whole matrix. A CT
exercises one module with its neighbours stubbed, an E2E the whole lifecycle; an INF goes deep into one mechanism
along the axis neither has - the kind of user code the mechanism has to cooperate with.

- **Its own Maven module** (`infra-tests`), depending on the starter and every plugin at the project's version, the
  way a user's build would.
- **A test `@SpringBootApplication` per mechanism**, only that mechanism switched on by profile, under
  `@SpringBootTest`. Nothing of the library is mocked; the user's side is test code - a `*BusinessService` and a
  repository per data-access variant (JDBC, JPA).
- **The matrix is chosen at run time, never compiled in**: the database by a system property (`-Ddb=`), the mechanism
  and the broker by a JUnit `@Tag` on the abstract base class (`-Dgroups=`) and Spring profiles. One run is one cell;
  CI runs every cell as its own job with `fail-fast: false`, in a workflow of its own. Only the selected cell's
  containers start, wired in the base class.
- **A scenario is written once, in a `*Verifier`** - a plain class without test annotations. Each variant is a thin
  `*InfraTest` that wires its beans into the verifier and delegates method by method, with its own `@DisplayName`
  (`KafkaJpaConsumerInfraTest`), so a scenario cannot drift between variants.
- **The context is shared, so the state is cleaned** in `@BeforeEach` and `@AfterEach`, database and broker alike -
  tables emptied, queues purged, topics truncated - or a message left from one case is delivered into the next.
  Variants in one context never share a queue or a topic, or their listeners compete for the same messages and every
  count is wrong.

**E2E** sits above a CT or an INF: the whole application, started as in production, driven only through its public
HTTP API.

- **Its own Maven module** (`e2e`), depending on the executable module's plain jar - so the Spring Boot executable is
  built with `<classifier>exec</classifier>`.
- **Two kinds of class**, the file name says which: **one per URL** (`PostAppointmentEndToEndTest`) answers whether
  that endpoint works through the whole stack, with the real database and security behind it; **one per flow**
  (`RegistrationFlowEndToEndTest`, `AppointmentCancelEndToEndTest`) answers whether a user gets through several URLs
  to the effect they end in.
- **The real `Application` class and configuration**, Postgres and Redis in singleton Testcontainers; one container
  set and one context shared by every class.
- **Only what lies outside the process is replaced, by an emulator where one exists** - SMTP by GreenMail behind a
  `@Primary` sender in a `@TestConfiguration`, a blob storage by Azurite in a container. None of the application's
  own beans is mocked (a CT fakes the same services as `InMemory*`).
- **Fixtures are built through the API as a user would** - register, confirm with the code read from the mail, create
  the profile - never by writing to the database. The test talks through a thin client keeping its own cookie jar and
  CSRF token.
- **A limit built for abuse** (a per-address rate limiter) that stands in the way is switched by profile for the run,
  not loosened in production code.
- **"Nothing was sent" is awaited too**, with `await().during(...)`.
- **A shared sink is filtered before it is read** - the mailbox by recipient and subject, since a registration letter
  lands in the same inbox.
- **The response body goes into the assertion message** (`assertThat(status).as(body)`), so a red run explains itself.
- **A negative case is always paired with the positive case of the same flow**: "this mails nobody" passes just as
  well when the whole feature is dead.

**E2E of a library.** A library has no production application and often no HTTP API. Its E2E module assembles a test
application from the starter with every mechanism on and drives it through the library's public API - the call that
publishes, the listener that consumes, the REST API the library exposes. The rest holds: real infrastructure, nothing
of the library mocked, fixtures through that API. What it covers that an INF does not is the lifecycle and its
failures.

## Where tests live and how they run

- **UT, MT, IT and CT live in `src/test` of the module they test**; Surefire picks them up by suffix in `mvn test`
  and `mvn verify`. One level runs alone by its suffix - `-Dtest='*ComponentTest'`, `-Dtest='*MultiThreadTest'`; an
  MT at the CT level answers to its own suffix, not the CT's.
- **What needs Docker** - an IT or MT against an engine, a CT, an INF, an E2E - **is skipped on a machine without
  Docker and fails in CI without it**: a runner that lost Docker must not report green with no CT in it, so
  `disabledWithoutDocker = true` is not used. The condition is a static method returning
  `DockerClientFactory.instance().isDockerAvailable() || System.getenv("CI") != null` - CI sets `CI`, so there the
  test runs and its container fails to start. JUnit's `@EnabledIf` is not `@Inherited`, so on a base class it does
  not reach the classes extending it: it is wrapped in a composed annotation of the test module, itself `@Inherited`
  (`@RequiresDocker`, carrying `@EnabledIf("com.example.test.Docker#availableOrCi")`). The base class - or the IT
  class where there is none - carries `@Testcontainers` and `@RequiresDocker`.
- **INF and the E2E of a library live in modules the root `<modules>` does not list**, out of the build that runs the
  levels above, and run against installed artifacts: `mvn install` of the library once, then `mvn verify` in the
  module - for INF with the cell chosen (`-Ddb=`, `-Dgroups=`, the profiles). **The E2E of an application is a module
  of the reactor**, built after the executable module.
- **A `@Tag("stress")` test** stays out of every default run through `<excludedGroups>stress</excludedGroups>` in the
  Surefire configuration - the tag alone excludes nothing - and is run on purpose with `-Dgroups=stress`.
- **No coverage percentage is a goal** - what a test asserts comes from the contract. The objective check is mutation
  testing: PIT (`pitest-maven`) on the core modules, run on purpose, not in every build. A surviving mutant is a case
  nobody listed - written as a test, or recorded as a mutation that changes nothing the contract promises.

## Naming

Method: `method_whenCondition_shouldResult` - lower camelCase, three parts joined by `_`.

`@DisplayName`: `"<level> <method>() when <condition> should <result>"` - normally the method name spelled out:

```java
@Test
@DisplayName("UT match() when several patterns match should take the most specific one")
void match_whenSeveralPatternsMatch_shouldTakeMostSpecificOne() { }
```

The display name may say more when the case needs opening up - a condition an identifier cannot carry, the reason
for the outcome. It may not be long: it is one line among hundreds in a run log, so the budget is another clause, not
another sentence. A name that wants a sentence usually means the test holds two cases.

For a constructor, and for a whole scenario in a CT, MT, INF or E2E, the first part is what happens, not a method:

```java
void constructor_whenPrefixIsNull_shouldThrowNullPointerException()
void context_whenExistingClockBean_shouldNotRegisterDefaultClock()                         // IT of an autoconfig
void save_whenCardIsUnknown_shouldAnswerSpecialistNotFound()                               // CT
void request_whenSameOrderIsPostedTwice_shouldBeAnsweredFromCacheWithoutReachingHandler()  // CT of a request filter
void save_whenCardVersionChangesMidway_shouldRetryAndCountTheRatingOnce()                  // MT
void consume_whenOperationFailsAndEventIsRedelivered_shouldNotRunOperationAgain()          // INF
void registration_whenUserConfirmsTheCode_shouldBeAbleToLogIn()                            // E2E of a flow
```

The result is **observable behaviour, not implementation**: `shouldReturnNullWithoutResolvingAnything`, not
`shouldNotCallResolver` - even when the assertion behind it is a `verify(resolver, never())`.

Helper classes in the test source root do **not** end in `Test` (`TestOrder`, `TestClock`): a reader takes a `*Test`
for a test class and looks for its cases. The prefix is fine - `Test*` and `*Test` are both in surefire's default
includes, but a class without a single `@Test` is simply not run.

## The shape of a test

```java
@Test
@DisplayName("UT getRoutes() when singletons are instantiated should return unmodifiable set")
void getRoutes_whenSingletonsAreInstantiated_shouldReturnUnmodifiableSet() {
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

- `// given` / `// when` / `// then` are labelled explicitly; when "when" and "then" cannot be separated,
  `// when / then`.
- The object under test is `tested` - an `@InjectMocks` field, or a local variable when the constructor depends on
  the case.
- One test, one scenario. Several `assertThat` calls are fine while they describe the same outcome.
- Data preparation goes into private helpers **at the bottom of the class**, after the tests; a test body keeps only
  what makes this case differ from the next one.
- Beyond the three labels, comments are written only where the reader would not follow otherwise - a non-obvious
  setup, a value meaning something its type does not show, a line atop the class saying what the file holds down. A
  comment restating the code, and javadoc on a test method, are noise: the display name already says what the case is.

Assertions are **AssertJ** (`assertThat`, `assertThatThrownBy` from `org.assertj.core.api.Assertions`). JUnit's
`assertEquals` / `assertTrue` and the import from `AssertionsForClassTypes`, still in a couple of old files, are
legacy and do not appear in new tests.

`@Nested` earns its place when one function has several independent groups of cases - the modes of a normalizer,
each with its own set. For a flat set of cases it is noise.

## Mocks and stubs

Mockito is for **collaborators that only answer** in this test: `@Mock` on the field, `@InjectMocks` on `tested`,
`@ExtendWith(MockitoExtension.class)` on the class. Strict stubs are on by default - a superfluous `when(...)` fails
the test, and that is correct.

A hand-written stub (`Recording*`, `Test*`, `Probe*` - a static nested class, or a class of its own next to the CT
base class when a module's CTs share it) is for when **state or behaviour** is needed: a call counter, an in-memory
store, "invoke the caller's callback". A mock whose `thenAnswer` holds logic always loses to such a class on
readability. It is named for what it does: `RecordingProcessor`, `RecordingEventChannel`, `InMemoryOrderRepository`.

Value objects (value types, metadata) are built, not mocked: `TestOrder.builder()` with sane defaults, one or two
fields overridden per case, living in its own module's test package.

"Nothing went here" is asserted with `verifyNoInteractions(...)` / `verify(x, never())`, not by the test's silence.

## What each level has to assert

**UT of a class:**
- every `Objects.requireNonNull` in the constructor - one test per argument, asserting the message text the code
  passes to it: a mechanical branch, whose body is its contract (see where an expectation comes from);
- every branch of every public method, including "found nothing" (`null`, empty collection) and the handling of a
  collaborator's exception;
- boundary inputs: `null`, empty string, blanks, empty collection;
- what the method does **not** do: did not touch the resolver, did not reach into the cache.

**MT:** both sides of the guarantee, as listed under MT.

**IT of an autoconfiguration:**
- the full set of beans when every required dependency is present - by interface and by concrete class;
- `@ConditionalOnMissingBean`: the user's own bean displaces the default (`hasSingleBean(Iface)` +
  `doesNotHaveBean(Default.class)`) - one test per `@Bean`;
- a missing required bean brings the context down (`assertThat(context).hasFailed()`);
- the conditions themselves: `@ConditionalOnWebApplication` by a run on a non-web `ApplicationContextRunner`;
- registration settings, not merely the bean's existence: filter order, url patterns, path prefix;
- the `@ConfigurationProperties` it binds: every property from its key, every default when the key is absent, and a
  context that fails to start on an invalid value (`@Validated`), naming the property;
- a decorator chain it assembles: the order on every wrapped bean, for every combination of decorators on and off.
  Behaviour does not show the order - a cache outside the metrics and one inside answer the same calls - so the test
  unwraps the chain (each decorator's `delegate`) and asserts the type at every level down to the implementation;
  and that two wrapped beans do not share a decorator instance.

**IT against a real external system:**
- every method of the contract against the real engine, the condition met and the condition missed;
- the negative half of a failed conditional write: not only that it threw, but that the stored row was left as it was;
- the absent case as the contract says: a missing key gives an empty `Optional`, not `null` and not an exception;
- nullable columns both ways - written as `NULL` by the write that leaves them empty, read back as `null` without
  blowing up;
- the invariants the contract states in prose: which write puts an expiry on a row, which clears an inherited one,
  which must not touch a column at all;
- concurrency the contract promises is an MT, not a case here.

The container is `@Container static`, shared by the class; the data is reset in `@BeforeEach` - what is cleaned
between cases is the state, not the engine.

**IT of the schema**, wherever it is migrated (Flyway, Liquibase, the library's own scripts):
- the migrations applied on the previous release's schema with data in it - not only on an empty database - and the
  data still read back afterwards;
- the migrated schema validated against the mapping (`spring.jpa.hibernate.ddl-auto=validate`), so an entity no
  longer matching its table fails here, not in production.

**A scheduled job** is run from a CT by calling its bean, like any other entry - a CT does not wait for the clock.
That it is scheduled at all - registered, with the cron or delay from its property - is asserted once by an IT over
the context, through `ScheduledTaskHolder.getScheduledTasks()`.

**CT:** the scenarios that cannot be assembled out of UTs:
- every use case of the module's API, from the call to what it stores, read back through the API, and the calls it
  makes to its neighbours;
- each use case against the edges it has to survive: not found, a duplicate, a version conflict, a failing
  neighbour.

Where the module's public API is itself a request filter (an idempotency library, say), its use cases are requests:
the whole request path, a repeat under the same key, a bypass on a non-idempotent endpoint, a missing header, an
error response.

**INF**, on every cell of the matrix:
- every guarantee of the mechanism, for every data-access variant: what the user writes and what the mechanism
  records are committed and rolled back together, through JDBC or JPA;
- redelivery as the guarantee says: the operation fails, the broker delivers again, and the event is processed again
  or skipped - whichever the chosen guarantee promises;
- duplicates: the same event twice, twice within one batch, and across batches;
- volume: a batch of tens and one of a thousand, so the paths that split work into chunks are reached;
- the configuration the documentation tells the user to use - the container factory, the acknowledgement mode - not
  one tuned for the test.

**E2E of an application, per URL:**
- the endpoint's success and its effect, read back through the API;
- the access rules on the real security filter chain: unauthenticated (401) and forbidden (403);
- one representative business error - a not found or a conflict - travelling the whole stack to the right status.
  The full matrix of statuses is not here: validation and status mapping are the controller's IT, business branches
  the CT.

**E2E of an application, per flow:**
- every user flow the API offers, from its first call to its last effect - what a follow-up call reads back, the
  letter that was sent;
- each flow in its positive case and its paired negative one;
- the effects that leave the process, read where they land (the mailbox), never from the application's own state;
- what crosses modules, which no CT sees: a rollback that has to undo the work of two modules together, and another
  module's listener that reacts only after the commit (`AFTER_COMMIT`) - firing on commit, silent on rollback.

**E2E of a library:**
- an event that goes the whole way, from the publishing call to the consuming listener, processed once;
- an outage: the broker stopped, the event retried, and delivered once the broker is back;
- recovery: an event stuck mid-send picked up again;
- retries exhausted into the dead letter queue, and the event delivered after a retry through the library's API;
- every clean-up the library schedules, run and its effect asserted;
- the library's own REST API, through its HTTP endpoints.

## Where an expectation comes from

A test asserts the contract, read from three sources:

- **the javadoc** - what the type and the method promise, including to callers that do not exist yet;
- **the project's design record** (an architecture section, a decisions log, ADRs, the agent instructions - wherever
  the project keeps its settled decisions and invariants) - often stricter than the javadoc, since it states what the
  construction relies on;
- **the call sites** - what callers inside the project depend on: which `null` they survive, which exception they
  catch, what they make of an empty result.

The body of the method under test is a source only for its mechanical branches - a `null` or empty input answered
with an early return, a `requireNonNull`, a sane fallback or normalization: the javadoc need not retell them, so the
body is their contract unless a source above says otherwise. A branch of business logic - deciding the outcome, the
status, what is stored or which exception is thrown - is never read off the body: that asserts the code against
itself.

A situation is a case only when the code decides something there or a source promises something there. Where the
code has no branch and no source says a word - an exception that simply propagates, the wording of a message the code
does not write itself - there is nothing to assert or list. Where a source promises what the code has no branch for,
it is a case all the same, and a red one.

The sources are not merged. A call site shows what a caller needs, not what the contract allows, and may lean on a
defect; the javadoc may promise more than anyone uses, and that part is contract all the same. Where the sources
agree, the case is asserted. Where they disagree or all stay silent, the case is listed in the report - *sources
disagree*, with what each says, or *inferred*, with the reading it rests on - and asserted only once the operator
decides, asked up front with the other questions.

## Tests and bugs that were found

Three rules. The first two look like opposites and are not: one is about a test asserting the defect, the other about
a test asserting the contract.

**A green test does not record a bug.** If an assertion passes only because the behaviour is broken, the test does
not enter the repository - it writes the defect down as if it were the contract, and the next reader has nothing left
to tell them apart by. Such a case is inverted, not deleted: assert what the contract says, and let it fail.

**A bug must still have its test, and that test is red.** This matters more than a green suite: a defect nobody wrote
a test for gets lost. The case is written against the contract the moment the bug is found, fails against today's
code, and goes green with the fix without being touched again. It is not disabled and not commented out - a
`@Disabled` test reports nothing, which is the whole problem it was meant to solve.

**The fix is not the test author's to make.** Production code is changed by the operator. Finding a bug means: write
the red test, write the finding into the project's open-items document (a plan, a spec, a backlog - whatever it keeps
open work in; in an application whose modules keep their own, the owning module's `ISSUES.md`) as its own item, with
the numbers of the run and a mark that it is reproduced, say it out loud - and stop there. A red suite whose red is
accounted for by a recorded item is a working state; a production file quietly patched to make it green is not.

Reproducing is done with a temporary probe class next to the tests (it prints statuses and counters to stdout). Once
the fact is written down, **the probe class is deleted** - what stays is the red test.

## Spring test plumbing

- **One test application or configuration per test module** (`*TestApplication`, or a CT `@Configuration` such as
  `<Module>ComponentConfig`), declaring what that module's tests need and nothing more. Only an E2E starts the
  production `Application`.
- **Constructor injection**: `@TestConstructor(autowireMode = AutowireMode.ALL)` on the base class, beans arriving
  through the test's constructor into `final` fields - not `@Autowired` fields.
- **One Awaitility helper per test module**, with named waits (`awaitAtMost(...)`, `awaitStable(...)`) and the
  timeouts and poll intervals defined once in it - not a `Duration.ofSeconds(15)` repeated in every test.

## Stability

- No `Thread.sleep` and no dependency on real time: time arrives through `Clock` (`TestClock`), and every
  asynchronous effect is awaited with Awaitility.
- Identifiers compared later are constants (`UUID.fromString("aaaaaaaa-...")`), not `UUID.randomUUID()`; a random
  UUID belongs exactly where "some other key" is meant. The exception is an E2E: its state is not cleaned between
  tests, so every case takes fresh identifiers (a unique email) - a constant id would meet the same id left by
  another case.
- Global state is cleaned in `@AfterEach` - `RequestContextHolder.resetRequestAttributes()`, for instance.
- Test order carries no meaning: every test prepares its own state.

## Formatting carried over from production code

Four-space indent, lines up to ~120 characters, explicit local types (no `var`), no Lombok, `"...%s"
.formatted(...)` rather than `String.format`, `byte [] bytes` with the space, enum comparison with the constant on
the left. Multi-line parameter lists align under the opening parenthesis; call arguments that do not fit wrap at an
indent of 8, one per line when there are more than two.

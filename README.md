# java-agent-skills

Skills for [Claude Code](https://code.claude.com) that carry a house style for Java and Spring projects - and
the procedures for restoring it once a refactor has left the code ahead of its tests and comments - plus the
traps of a library no model has seen.

The conventions are opinionated on purpose. The assumed stack is Java with Maven, Spring, JUnit 5, Mockito,
AssertJ, Testcontainers and jacoco; `spring-conventions` adds Spring MVC, Spring Security and
springdoc-openapi.

## Install

```
/plugin marketplace add dmitriy-iliyov/java-agent-skills
/plugin install java-conventions@dmitriy-iliyov-skills
/plugin install spring-conventions@dmitriy-iliyov-skills
/plugin install oncebox@dmitriy-iliyov-skills
```

Each plugin is installed on its own. `java-conventions` holds for any Java project, `spring-conventions` for a
Spring web application, `oncebox` for a Spring Boot service that publishes or consumes events through it.

## What you get

### `java-conventions`

Four skills, in two pairs. The `style` one says what the result must look like; the `refactor` one says what
to do and in what order.

**`javadoc-style`** - what earns a comment and what never does. Stops docs from restating the method name,
and keeps out the four genres that creep in while the code is written: a defence of an absence, thinking out
loud, a `TODO`, and the name of a test that supposedly holds the statement down.

**`javadoc-refactor`** - a pass over comments that describe types which have moved. Two passes in a fixed
order, the greps that find each kind of stale comment cheaply, and a verification step, because a broken
`{@link}` compiles happily and only the javadoc tool catches it.

**`test-style`** - three levels (`UT` / `IT` / `CT`), what is alive and what is stubbed at each, how a case is
named, what may be mocked and what may not, and what every level has to assert. Includes the rule most worth
having: a green test never records a bug.

**`test-refactor`** - getting a module's suite back to green after a refactor, module by module, contract
first. Sorts every red test into "the test is stale" and "the code is wrong", and treats them in opposite
ways - the second stays red and becomes a finding, rather than being laundered into a passing test.

### `spring-conventions`

Three standards, without a procedure yet.

**`controller-style`** - Spring MVC controllers. A controller method is one call to the service; URLs,
parameters, bodies and statuses have one shape each, and input is validated by annotations. Covers REST only
for now.

**`controller-advice-style`** - how an exception becomes an error response. Every error is a `ProblemDetail`
whose status, type and title are decided in one factory; the advice hands security exceptions back to Spring
Security, so that an anonymous caller gets `401` rather than `403` or `500`; one advice per module, ordered
ahead of the global one.

**`swagger-style`** - springdoc annotations on a controller and its records: what each endpoint and field
carries, how the texts are worded, which status codes are listed and which never are. Names the annotations
that render a wrong schema without an error, and checks the result in the generated `/v3/api-docs` rather
than in the source.

### `oncebox`

**`oncebox`** - the [oncebox](https://github.com/dmitriy-iliyov/oncebox) transactional outbox for Spring Boot,
Kafka and RabbitMQ. Not a restatement of its README but what the README leaves unsaid: the three modules a
working build needs, when the library does not fit, the settings that fail the context at startup, and the
ones that break delivery without an error - MySQL deadlocks under `REPEATABLE READ`, a thread pool sized for
one event type, Kafka committing offsets before the operation ran, one cache name shared by several services.
Also fires before an outbox poller is written by hand.

## Using them

Claude Code loads a skill on its own when the task matches its description. To reach for one directly:

```
/java-conventions:javadoc-style
/java-conventions:test-refactor
/spring-conventions:controller-style
/oncebox:oncebox
```

## Making them yours

Every rule in the conventions is a claim about how code should look, and a project is free to disagree with
any of them - that is why each `style` skill says a rule broken is "a reason to change the rule here
deliberately" rather than a law. Fork the repository, edit the `SKILL.md`, and point the marketplace at your copy.

A rule that mentions a module, a type or a test by name belongs to one codebase, not here: such a skill stays
in its own repository under `.claude/skills/`.

## Releasing

Consumers detect an update by the `version` field in `plugins/<plugin>/.claude-plugin/plugin.json`. Bump it
whenever a skill of that plugin changes, or `/plugin update` will not see the new edition.

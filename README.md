# claude-skills

Skills for [Claude Code](https://code.claude.com) that carry a house style for Java projects - and the
procedures for restoring it once a refactor has left the code ahead of its tests and comments.

They are opinionated on purpose. The assumed stack is Java with Maven, Spring, JUnit 5, Mockito, AssertJ,
Testcontainers and jacoco.

## Install

```
/plugin marketplace add dmitriy-iliyov/claude-skills
/plugin install java-conventions@dmitriy-iliyov-skills
```

## What you get

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

## Using them

Claude Code loads a skill on its own when the task matches its description. To reach for one directly:

```
/java-conventions:javadoc-style
/java-conventions:test-refactor
```

## Making them yours

Every rule in these skills is a claim about how code should look, and a project is free to disagree with any
of them - that is why each skill says its rules are "a reason to change the rule here deliberately" rather
than a law. Fork the repository, edit the `SKILL.md`, and point the marketplace at your copy.

A rule that mentions a module, a type or a test by name belongs to one codebase, not here: such a skill stays
in its own repository under `.claude/skills/`.

## Releasing

Consumers detect an update by the `version` field in `plugins/java-conventions/.claude-plugin/plugin.json`.
Bump it whenever a skill changes, or `/plugin update` will not see the new edition.

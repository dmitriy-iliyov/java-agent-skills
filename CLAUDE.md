# claude-skills

A Claude Code plugin marketplace. Everything in it is prose meant to be read by a model - there is no code
to build, no tests to run and no dependencies to install.

## Layout

```
.claude-plugin/marketplace.json      the catalogue - every plugin is listed here
plugins/<plugin>/
    .claude-plugin/plugin.json       name, description, version
    skills/<skill>/SKILL.md          the skill itself
```

Adding a skill to an existing plugin: write the `SKILL.md`, then bump `version` in that plugin's
`plugin.json`. Adding a plugin: write both manifests and register it in `marketplace.json` - a plugin
missing from the catalogue cannot be installed, however complete its own files are.

**The version bump is not optional.** Consumers detect an edition by `version` in `plugin.json`. Without a
bump `/plugin update` sees nothing and the changed skill never reaches anyone who already installed it.

## What belongs here

A rule that holds for Java projects in general. A rule that names a module, a type or a test of one codebase
belongs to that codebase, under its own `.claude/skills/`.

Skills here come in three shapes, one example of each already present:

- a **standard** - what the result has to look like (`javadoc-style`, `test-style`);
- a **procedure** - what to do and in what order (`javadoc-refactor`, `test-refactor`);
- **knowledge of an external thing** - the traps of a library no model has seen (`oncebox`).

A standard and its procedure stay separate skills and link to each other. Neither absorbs the other: one is
read while judging a result, the other while working through a pass, and they are needed at different
moments.

## Writing a skill

English throughout, as the skills themselves require of code, javadoc and message text.

**The `description` decides whether the skill is ever read, and nothing else does.** It names the *task* in
the words of someone who does not know this skill exists - a skill about a library that triggers only on
that library's name will never fire, because whoever needs it has not heard the name either. Error text the
reader would paste, and file contents that give the situation away, are stronger triggers than a topic.

It also states what the skill is **not** for. A negative clause is the only thing that keeps two skills
whose subjects share a word from answering for each other.

**The body assumes it has already been chosen.** No restating of the description and no case for the skill's
own existence - what to do, in the order it is done.

Prose rules, the ones the skills apply to javadoc:

- lines wrap at about 120 characters;
- a dash inside English text is a plain hyphen with spaces (` - `), never an em dash;
- a claim is concrete - the property name, the error as it is printed, the command as it is run;
- a rule that is a judgement says so instead of being dressed as a law.

That last one holds for this file too. Every skill here states that breaking one of its rules is "either
wrong, or a reason to change the rule here deliberately".

## Commits

`type(what is true after the change)` - lowercase, no colon, the parentheses carrying the description
itself. The types in use are `refactor`, `fix`, `add` and `docs`.

The subject states the **resulting state**, in the present tense, subject first - not the action that was
performed. Two such statements are joined with ` & `:

```
refactor(null is a fact of the record & the core stops asking a format about it)
fix(a response past its expiry is no longer replayed & the manager says why a lookup came back empty)
add(starter binds idempify.* into the bottom configuration layer)
```

The body is prose wrapped like everything else: what was wrong before, what the change makes true, and why
the obvious alternative was not taken. It names the types it touched, and its last paragraph accounts for
what moved with them - tests, docs, comments.

The two commits already in this repository predate the convention.

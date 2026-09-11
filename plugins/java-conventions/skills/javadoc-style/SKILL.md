---
name: javadoc-style
description: The javadoc style for a Java project - what earns a comment and what never does, where a type, method or constructor gets one and where nothing goes at all, and the formatting rules. Load it before writing or rewriting javadoc, and whenever judging whether an existing comment is worth its place. The procedure for auditing existing javadoc against the code it describes is the separate `javadoc-refactor` skill.
---

# Javadoc style

House rules for javadoc in `src/main`. A comment that breaks one of them is either wrong, or a reason to
change the rule here deliberately.

Code, javadoc and message text are written in English.

## What a comment is for

Javadoc says what the signature cannot: **the contract, the meaning of `null`, the invariant behind a value**
and - rarely - **the reason for a decision**.

Restating the name of a method is worse than writing nothing. The norm is two or three sentences; a second
paragraph has to be earned.

A reason earns its place only where the code without it reads as a **mistake or as unfinished work**. That is
the whole test. "Publishing happens after the transaction commits, because until then the operation does not
exist" passes it - the ordering looks arbitrary until you know. "A record rather than a class because that is
all it is" does not.

## What never goes in

The middle ones are what creep in while the code is being written; the last is what creeps in while it is
being tested.

**1. The name, spelled out.** `getTtl()` returning "the ttl" documents nothing.

**2. A defence of an absence.** Why the type has no further parameter, why a method was not added, why an
alternative nobody proposed was not chosen - none of it belongs in the doc:

```java
// no
 * The declared type of the result is not among the components on purpose. It travels on Operation but
 * stops here: it belongs to the call site, which every entry that reads a result back already holds, so a
 * column for it would only duplicate what the caller brought.

// yes
 * The declared type of the result is not among the components: it travels on {@link Operation} and comes
 * from the call site on every read.
```

**3. Thinking out loud while developing.** A sequential argument about what type a parameter will or will not
be, a walk through the options that were considered, "at first this was a `Class`, then it turned out":

```java
// no
 * {@code Class} was the obvious carrier and the wrong one: it has no room for type arguments, so a method
 * returning {@code List<Order>} arrived at the deserializer as a bare {@code List} and came back as a list
 * of maps. The information was never lost to erasure - it was lost to the descriptor this library chose.

// yes
 * The type a stored result is read back into, carried as a {@link Type} so that a parameterized return type
 * keeps its type arguments: a method returning {@code List<Order>} is replayed as a list of orders and not
 * as a list of maps.
```

The doc describes **what came out**, not how it was arrived at. What reached the code is written as a
statement; the road to it stays in the git history and in the project's open-items document
(a plan, a spec, a backlog - whatever this project keeps open work in).

**4. Work still to be done.** A doubt, a "decide later", a `TODO` - none of it is javadoc. Javadoc is the
published contract of a type, and a note to the author is not part of a contract.

A `TODO` or a `FIXME` is legitimate, just not in this form: it goes in an ordinary `//` comment at the line it
concerns, where it reads as a marker rather than as documentation and is found by a grep for markers. A
question big enough to need arguing goes to that document instead.

**5. A test, named.** Javadoc never says which test holds a statement down. The reader of the published API
does not have that class - test sources are not shipped - and `{@link}` cannot reach `src/test` from
`src/main` anyway, so the name arrives as unresolvable `{@code}` that no tool will notice going stale when the
test is renamed.

It also puts the guarantee in the wrong place. Where an invariant holds **by construction**, say what makes it
hold and the sentence becomes stronger for it; where it holds only because something enforces it, that
belongs in the test's own `@DisplayName`, not in the contract of the type:

```java
// no
 * The config classes read these strings for their own DEFAULT_* constants, so a default is written here
 * once; that the two agree is what {@code PropertyDefaultsUnitTest} checks.

// yes
 * The config classes read these strings for their own DEFAULT_* constants, so a default is written here
 * once and answered the same way from both sides - the two cannot say different things.
```

The same goes for the coverage a type has, the run that confirmed a fix, and the plan item that tracks it.

## Where a comment goes

| Where | Rule |
|---|---|
| **Interface, enum, annotation** | effectively always - it is the contract, and there is no body to read instead |
| **Class** | almost always if it is part of the API or carries a non-obvious decision |
| **Method** | only when there is something to say beyond the name |
| **Constructor** | only if it does something unexpected with an argument |
| **Getter, builder setter, one-liner** | nothing at all |
| **Private trivial helper** | nothing at all |

The exemptions are not a licence to be terse elsewhere - they are what keeps the remaining comments visible.
An obvious builder setter (`enabled(boolean)`, `handler(...)`) gets one short line or, better, nothing; a
multi-line paragraph on it is noise that hides the places where a doc was actually needed.

A setter earns a line only for a non-obvious contract: which combinations it refuses, what a `null` means,
what gets substituted when it is not called.

## Formatting

- `<p>` on a line of its own between paragraphs; no `</p>`.
- References through `{@link}` and `{@code}`; `{@link}` for a type or member that exists, `{@code}` for
  literals, property names and anything not resolvable.
- A dash inside English text is a plain hyphen with spaces (` - `), not an em dash.
- Lines wrap at about 120 characters, like the code.
- `@param` is **all or nothing**: document one and you document the rest, `<T>` included. Getters and setters
  are exempt. The order follows the signature.
- `@return` describes the value, including whether it can be `null` and what `null` then means.
- `@throws` for an exception a caller is expected to act on, including an unchecked one.
- `@see` points at the type that explains the context.
- `@deprecated` and `@Deprecated(since, forRemoval)` come together.
- `{@inheritDoc}` when an override only adds a clause - typically a new `@throws`.

## Never document the intention

Javadoc describes the code as it is, not as it is meant to become. Nothing here is invented: a `null` rule
still undecided is left out of the doc rather than guessed at, and a contract nobody has settled is not
settled by writing it down.

The other half of the same rule: **a defect found while writing or reviewing a doc is not a documentation
task.** It does not get written into the javadoc - neither as the intended behaviour, which would hide it, nor
as the broken behaviour, which would enshrine it. The doc stays silent on the contested point and describes
what is not in doubt.

The defect itself is **reported** - always, and in the same breath as the doc change. Then it goes where
defects go in this project: the open-items document. A doc pass that quietly leaves a bug behind it has done
the worse half of the job.

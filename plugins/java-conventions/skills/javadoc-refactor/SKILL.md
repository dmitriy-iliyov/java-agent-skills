---
name: javadoc-refactor
description: The procedure for bringing a project's javadoc back in line with code that has moved - the file inventory, two passes in a fixed order (distortions and absences first, restatements second), how to find each kind of stale comment cheaply, how to verify the result, and how to report it. Use it after a refactor has left comments describing types that no longer exist, when asked to go over the javadoc of a module or of a set of changed files, or when judging whether existing comments still describe what the code does.
---

# Javadoc refactor pass

A pass over code whose production types have moved and whose comments have not caught up. Stale javadoc is
worse than none: it is read as a contract, and unlike a stale test it fails nothing.

Write the comments in the house style: see the `javadoc-style` skill
([../javadoc-style/SKILL.md](../javadoc-style/SKILL.md)) - what earns a comment, where one goes, what never
goes in, formatting. This document says *what to do and in what order*; that one says *what the result must
look like*.

## Unit of work

The inventory is **the files the refactor touched**, not the whole tree - and `src/main` only, since tests
carry their contract in `@DisplayName` rather than in javadoc:

```bash
git status --porcelain | awk '{print $NF}' | grep 'src/main/.*\.java$' | sort -u \
    | while read f; do [ -f "$f" ] && echo "$f"; done > files.txt
```

Filtering by `-f` matters: a rename shows both sides, and the deleted side is still in the listing.

Then classify before reading. Which files are interfaces or enums, and which have no type comment at all, is
one cheap query and it decides most of the work:

```bash
while read f; do
    kind=$(grep -m1 -oE "public (interface|enum|@interface|final class|abstract class|class|record)" "$f")
    printf "%-58s %-20s blocks=%s\n" "$f" "$kind" "$(grep -c '^\s*/\*\*' "$f")"
done < files.txt | grep -E "interface|enum"
```

## The two passes

The order is fixed. Pass 2 is the cheap half, and doing it first buries the half that matters in a diff too
large to review.

### Pass 1 - distortions and absences

A comment is a **distortion** when it contradicts the code. Three kinds, in ascending cost:

1. **A dead reference** - a renamed or deleted type still named in the text. Cheap, and worth doing across
   every file at once before reading any of them:

   ```bash
   while read f; do
       awk -v F="$f" '/\/\*\*/,/\*\//{ if ($0 ~ /OldTypeName|RemovedConfig/) print F": "$0 }' "$f"
   done < files.txt
   ```

2. **A tag out of step with the signature** - `@return` promising a component the type no longer carries, a
   missing `@param` after an argument was added, `@param` order that no longer follows the signature. Found
   by reading the tags against the declaration, no code needed.

3. **A claim the code does not honour** - the expensive kind, needing both the doc and the code. An interface
   saying "reading is not part of this contract" while declaring a read method; a properties class naming a
   property that has since been renamed; a switch documented as irreversible after the mechanism that made it
   so was deleted. Verify every load-bearing claim against the implementation - a sentence that sounds
   authoritative is exactly the one to check.

An **absence** is a missing type comment on an interface, an enum or an annotation - nested ones included, and
including a type whose methods are documented but whose own contract is not. Classes are not part of this
pass unless they are asked for: the rule for them is "almost always", which is a judgement, while for an
interface it is effectively a rule.

### Pass 2 - restatements

Only once pass 1 is closed: strip the comments that repeat the signature - getters, builder setters,
one-liners, private trivial helpers - and shorten the ones on obvious builder setters to a single line or to
nothing. What survives is a setter documenting a real constraint: a refused combination, the meaning of
`null`, what is substituted when it is not called.

In the same pass, take out the names that point outside the published API - a test class, a run, a recorded
item. `grep -n "UnitTest\|IntegrationTest\|ComponentTest" -r src/main` finds most of them in one go. A
sentence that leaned on a test for its authority is restated so that it stands on the construction instead.

## What the pass must not do

**Do not document the intention.** Where a doc and the code disagree, neither the intended behaviour nor the
broken behaviour goes into the comment. The doc stays silent on the contested point.

**Do not swallow a defect.** Anything found while reading - a guard with the wrong operator, a
`requireNonNull` naming the wrong parameter, a method name left over from a rename - is **reported** and
written into the project's open-items document (a plan, a spec, a backlog - whatever this project keeps open
work in). It is not fixed inside a javadoc pass, and it is not left unmentioned. Code
findings and doc findings are reported separately, because only the second are what this pass changed.

## Verification

Compilation catches an unbalanced comment and nothing else. A broken `{@link}` is caught only by the javadoc
tool, so run it for every module touched:

```bash
./mvnw -o -pl <module> clean compile
./mvnw -o -pl <module> javadoc:javadoc
```

Both must be `BUILD SUCCESS`. Hence a rule for the pass itself: never introduce a `{@link}` to a type you
have not confirmed exists and is imported or fully qualified.

If a module's tests do not compile, `install -DskipTests` still fails on them - `skipTests` skips running, not
compiling. Use `-Dmaven.test.skip=true` when a downstream module needs this one installed.

## Reporting

Report by file, and for a class list the affected members as `Class#method`.

Group the findings by kind - **distortion**, **forbidden genre**, **absence** - and keep code findings in a
section of their own. The grouping is not cosmetic: a distortion usually means that document is stale in the
same place, so each one is a prompt to check it, while a forbidden-genre finding is only ever about the
comment.

Close by naming the recorded items the pass closed and the ones it only narrowed, so the document can be
trimmed in one go afterwards rather than drifting again.

## Where the findings go

A defect confirmed by a run is written into the project's open-items document as its own item, marked as
reproduced, with the concrete numbers. A defect found by reading is written the same way, without the mark.
Nothing is fixed silently outside the comment the pass came for, and no recorded item is marked with a
tick - a finished item is deleted from the document entirely.

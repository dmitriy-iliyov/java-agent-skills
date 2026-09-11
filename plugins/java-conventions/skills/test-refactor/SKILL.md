---
name: test-refactor
description: The procedure for bringing a multi-module Maven/Java project's tests back to green and up to coverage after a refactor - module by module, package by package, contract first. Use it when a refactor has left tests failing or not compiling, when a module's coverage has to be raised deliberately rather than opportunistically, or when asked to "go through the modules and sort out the tests".
---

# Test refactor pass

A pass over a project whose production code has moved and whose tests have not caught up. The order is
fixed, the gate between steps is fixed, and the treatment of a red test depends on how much of its class is
red.

Write tests in the house style: see the `test-style` skill
([../test-style/SKILL.md](../test-style/SKILL.md)) - levels, naming, the given/when/then shape,
what may be mocked, what each level has to assert. This document says *what to do and in what order*; that
one says *what the result must look like*.

## Unit of work

One **module** at a time, in dependency order (a module nobody depends on goes last).

Inside a module, a **package is its own step** when it is a meaningful piece of the module - it has its own
extension point, its own exceptions, its own value types. A package that is only utilities, containers or
DTOs is not a step of its own: it is handled together with the module's root package.

Do not touch a module that is not the current step. A defect found in a neighbouring module is written down,
not fixed in passing.

## Order within one step

### 1. Read the contract first

Before opening a single test:

- read `src/main` of the package - the interfaces, then the implementations behind them;
- read the javadoc as the contract it claims to be, and note where the code and the javadoc disagree;
- find out **how the components are actually used**: grep for the call sites across all modules, not only
  inside this one. A type's contract is what its callers depend on, not what its own test says about it.

The current contract is the source of truth. Never reconstruct it from an existing test - that test is
exactly what is suspected of being out of date.

### 2. Run the module's tests

```bash
./mvnw -pl <module> test                   # tests of one module
./mvnw -pl <module> -am test               # ... together with the modules it depends on
./mvnw -pl <module> -Dtest=SomeUnitTest test
./mvnw -pl <module> verify                 # tests + the jacoco report
```

Read the numbers, not the impression: how many ran, how many failed, how many errored, and - separately -
whether the test source root compiles at all.

**A test source root that does not compile is the first thing to fix.** Until it does, "red" and "green" mean
nothing and no item below can be confirmed or closed.

A class that does not compile has not said how red it is - it has said nothing at all, and the count step 4
needs cannot be taken from it. Restore compilation first, mechanically and case-preservingly: swap the
changed signature, move the file into its type's package, rename it after the type it now tests. Only a class
written against something that no longer exists at all - a deleted type, a column the row no longer has - has
lost its intent along with its subject, and that one is rewritten in full there and then. Then run, and apply
step 4's threshold to what actually fails.

What this avoids: a class of twenty sound cases whose two helpers carry an old signature. Counted as "100%
red" it would be rewritten wholesale, and twenty good tests would be thrown away to repair two lines.

While a large refactor is in flight, a module built without `-am` picks the neighbouring modules out of
`~/.m2` and is therefore compiled against a stale snapshot. Either install the dependency again
(`./mvnw -pl <dependency-module> install -DskipTests`) or always run with `-am`.

### 3. The gate

Green **and** coverage above 95% → the step is done, move to the next package or module.

Green here means "nothing red that is not accounted for". A test left red on purpose - it asserts the
contract, the code does not honour it yet, and the project's open-items document (a plan, a spec, a backlog -
whatever this project keeps open work in) carries the item - does not hold the step open: it is the finding,
handed over. What holds the step open is red nobody has explained.

The 95% is a signal, not a target. When the rest of it is out of reach for a reason - a defensive branch an
earlier validation already rules out, a `catch` that cannot fire, a type whose entire body is getters and a
`toString` - the step closes below the number, with the reason written into that document and named in
the report. **What must never happen is padding.** A test that calls a getter to watch it hand back what the
constructor was given asserts nothing, costs a file to keep in step with the type, and still moves the
percentage - which is precisely why the percentage stops meaning anything once such tests are in it. A
module at 91% whose missing nine points are named is in better shape than one at 96% where five of them are
noise.

Coverage is read from the jacoco report produced by `verify`, under
`<module>/target/site/jacoco/` (`jacoco.csv` for the numbers, `index.html` to see what is missed). Both
instructions and branches count.

Anything else → continue with step 4.

### 4. Red tests first, and only them

Coverage is not raised on top of a red module: a new test written next to a failing one inherits its wrong
picture of the contract.

Every red test is first sorted into one of two kinds, because they are treated in opposite ways:

- **the test is out of date** - the contract moved and the test did not follow. This is what the rest of the
  step is about: repair it or rewrite it;
- **the code is wrong** - the test states what the contract promises, and the code does not deliver it. This
  one is not repaired at all. It stays red, the finding goes into the open-items document, and the fix is the
  operator's.

Telling them apart is a question about the contract, never about which is easier to make green: does today's
`src/main` behaviour match what the interface and its javadoc promise? If it does, the test is stale. If it
does not, the test is right and the code is the defect.

For each red test class of the first kind, compute the **share of red tests in that class**:

| Share | What to do |
|---|---|
| around 30% or more | **rewrite the whole class**, from the contract, in the house style |
| below that | **remove the red tests one at a time**, rewriting each one's case against the current contract; leave the green ones alone |

Rewriting a case means keeping the intent and re-deriving the expectation: what was this test trying to
protect, and what does the current contract say about that same situation? A case that has lost its subject
entirely (the branch is gone, the type no longer exists) is deleted rather than reinterpreted, and its
disappearance is reported.

What the pass must not do in this step:

- **The production code is not adjusted to make a test pass.** Production code is changed by the operator,
  not by this pass. If the test is right and the code is wrong, that is a finding to report, not a file to
  repair.
- **The test is written anyway, and left red.** A bug with no test is a bug that gets lost, so the case is
  written against the contract, fails against today's code, and goes green with the fix without being
  touched again. It is not disabled and not commented out: what explains the red is the recorded item, not a
  marker in the test.
- **An assertion is not weakened to reach green.** A test that no longer asserts anything is worse than a red
  one - it is a red one that stopped reporting.
- **A green test is not left standing on broken behaviour.** An assertion that passes only because the
  behaviour is wrong has written the defect down as the contract. It is inverted to assert the contract and
  left red, and the finding goes into the open-items document with the numbers of the run.
- **A decision that belongs to the owner of the project is not taken silently.** When the right fix requires
  choosing between architectures or behaviours, bring the options and wait.

### 5. Cover what has no test

Only once the module is green:

- every type in the package without a `UT` - starting with those that carry behaviour, not with the
  containers. A type with no behaviour at all earns a `UT` only where something in it can actually be wrong:
  a builder setter that normalises or rejects, a constructor guard, an `equals`/`hashCode` something else
  relies on. Getters that hand back what was passed in are not one of those places;
- every branch the jacoco report shows as missed, or an explicit statement of why it is unreachable
  (a defensive `throw` that an earlier validation already rules out, a `catch` that cannot fire). Unreachable
  branches are recorded in the open-items document, not chased with artificial tests;
- every bean the module contributes - a line in the `IT` of its autoconfiguration: present in the full
  context, displaced by a user's own bean, and the context failing without each required foreign dependency;
- for a module that speaks to a real external system, an `IT` on Testcontainers
  per method of the store contract - both outcomes of every conditional write, the absent case, the nullable
  columns. The guarantees such a module exists for cannot be covered by a UT over a mocked driver, so a green
  UT there says nothing about them.

### 6. Lock it in with a component test

The step ends with a `CT` - the module working as a whole, or a package that is a meaningful piece of it (not
a utility or container package). Rewrite the existing one against the new contract, fix it, or write it if
there is none.

The rule that makes the CT worth writing: **components of your own module are never mocked** - only the
neighbouring modules are. A CT with its own module stubbed out asserts a guess instead of the wiring.

The scenarios to keep: the whole path end to end, a repeat under the same key, the case that bypasses the
mechanism entirely, a missing input, an error outcome.

### 7. Report before moving on

Report by file, and for a test class list the affected cases as `Class#method`.

Per step, briefly: red → green numbers, coverage before and after, which classes were rewritten in full and
which were repaired case by case, which cases were deleted with their subject, and what was found but not
fixed because it belongs to another module or needs a decision.

Separately and by name: **the tests left red on purpose** - which case, which contract it holds to, which
recorded item explains it. That list is the handover; without it a red suite is indistinguishable from an
unfinished one.

## Where the findings go

A defect confirmed by a run is written into the project's open-items document as its own item, marked as
reproduced, with the concrete numbers. A defect found by reading is written the same way, without the mark.
Nothing is fixed silently in a module that is not the current step, and no recorded item is marked with a
tick - a finished item is deleted from the document entirely.

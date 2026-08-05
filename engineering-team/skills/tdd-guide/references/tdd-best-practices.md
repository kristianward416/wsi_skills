# TDD Best Practices

This document covers the methodology behind `tdd-guide`'s workflow: the
discipline of red-green-refactor, when test-first beats test-after (and
when it doesn't), the anti-patterns that quietly destroy the value of a
test suite, and triangulation as a technique for driving out a general
implementation from specific examples.

## 1. The red-green-refactor discipline

TDD is not "write tests." It is a tight loop with three distinct, ordered
phases, and skipping or blurring them is the most common way teams end up
with TDD in name only.

### RED — write a test that fails, on purpose, for the right reason

Before any production code changes, write the smallest test that expresses
one new piece of behavior. Run it and **watch it fail**. This step is
frequently skipped by developers who "know" the test will fail — but the
failure itself is informative:

- If it fails with an import error or syntax error, the test isn't
  actually exercising anything yet.
- If it fails with the assertion you expected (e.g. `expected 42, got
  undefined`), you've confirmed the test can detect the missing behavior.
- If it unexpectedly *passes*, either the behavior already exists or the
  test is not asserting what you think it is — investigate before
  proceeding.

A test suite where every test has never been seen to fail is a test suite
nobody can trust.

### GREEN — the simplest thing that could possibly work

Write the minimum implementation to pass the failing test — no more. This
often means hard-coding a value, using an inefficient algorithm, or
special-casing. That is fine and expected; the design pressure to
generalize comes from *later* tests failing under the naive
implementation, not from anticipating them now. Resist implementing
features "while you're in there" that no test requires yet — untested
code paths are a liability, not a bonus.

### REFACTOR — improve structure, not behavior

Only refactor with a fully green suite. Refactoring changes *how* the code
is organized without changing *what* it does — extract a function, rename
for clarity, remove duplication, replace a hard-coded value with a real
computation once triangulation demands it (see below). Run the suite
after every small refactor step, not just at the end. If the suite goes
red during refactor, the change was a behavior change in disguise —
revert and redo it as a smaller step.

### Why order matters

Writing production code before its test ("test-after," see below) or
refactoring while red both break the feedback loop that makes TDD safe.
The value of TDD comes specifically from the tight red → green → refactor
sequence: it guarantees every line of production code exists because a
test demanded it, and every test has been proven capable of failing.

## 2. Test-first vs. test-after

| | Test-first (TDD) | Test-after |
|---|---|---|
| Test drives design | Yes — interfaces emerge from how they're easiest to test | No — tests conform to whatever design already exists |
| Confidence test can fail | High — you watched it fail | Low — never observed failing, may be vacuously true |
| Coverage of edge cases | Tends to be higher — you think through cases before coding | Tends to mirror only the cases the author remembered while coding |
| Speed for well-understood problems | Slower up front | Faster up front |
| Speed for exploratory/spike work | Can slow down exploration | Better fit — throwaway code doesn't need TDD rigor |
| Risk of testing implementation details | Lower — test is written against a not-yet-written implementation, so it can only describe behavior | Higher — easy to write a test that mirrors the code's internals because the code is already visible |

**When test-after is a reasonable choice:**
- Exploratory spikes whose code will be thrown away.
- Adding characterization tests to legacy code with no tests, purely to
  create a safety net before refactoring (write the test to match current
  behavior, then improve).
- Tests added retroactively for a bug fix, where the test *is* written
  first relative to the fix (red for the bug, green after the fix) — this
  is actually still test-first for that specific change.

**When test-after quietly fails:** the most common failure mode is that
"we'll add tests after" becomes "we added tests that pass" without ever
confirming they can fail — i.e., tests that assert `true == true` in
substance. If adopting test-after, at minimum mutate the implementation
temporarily to confirm each test can fail before trusting it.

## 3. Common anti-patterns

### Testing implementation, not behavior

A test that breaks whenever you refactor internals — without any change in
observable behavior — is testing implementation, not behavior.

```python
# Anti-pattern: asserts on a private internal call
def test_calculates_total():
    calc = OrderCalculator(order)
    calc.calculate()
    assert calc._subtotal_cache == 100  # internal detail

# Better: asserts on observable behavior
def test_calculates_total():
    calc = OrderCalculator(order)
    assert calc.calculate() == 100
```

Symptoms this anti-pattern is present:
- Renaming a private method or variable breaks tests.
- Tests reach into `_private` attributes or call internal helpers
  directly.
- Refactor phase routinely turns the suite red even though behavior is
  unchanged.

Fix: assert only on public inputs/outputs — return values, thrown
exceptions, observable side effects (a written file, a sent request, a
persisted record) — never on internal state or call sequences unless the
call sequence *is* the contract (e.g., verifying a third-party API is
called with x, when that call is the entire point of the unit).

### Overmocking

Mocking every collaborator turns a test into a restatement of the
implementation rather than a check of behavior.

```javascript
// Anti-pattern: mocks so much the test only checks wiring
test('processes order', () => {
  const validate = jest.fn().mockReturnValue(true);
  const calculate = jest.fn().mockReturnValue(100);
  const save = jest.fn();
  processOrder({ validate, calculate, save }, order);
  expect(validate).toHaveBeenCalled();
  expect(calculate).toHaveBeenCalled();
  expect(save).toHaveBeenCalledWith(100);
});
// This test passes even if validate/calculate/save are all broken —
// it only proves processOrder calls its collaborators in some order.

// Better: mock only true external boundaries (network, disk, clock),
// use real objects for in-process collaborators
test('processes order', () => {
  const save = jest.fn();
  processOrder({ save }, realOrder);
  expect(save).toHaveBeenCalledWith(expectedTotal);
});
```

Guidance:
- Mock at architectural boundaries (network calls, databases, filesystem,
  system clock, randomness) — not at every function/class boundary.
- If a test needs five mocks to run, that's a signal the unit under test
  has too many collaborators — a design problem, not a testing problem.
- Prefer real, small, fast collaborators (in-memory implementations, pure
  functions) over mocks whenever the collaborator is fast and
  deterministic.
- A mock that never fails a test (because it's configured to always
  succeed) is decoration, not a test — question whether it belongs.

### Other anti-patterns worth naming

- **Assertion roulette** — a test with many assertions and no messages,
  so a failure gives no clue which one broke. Prefer one behavior per
  test, or descriptive assertion messages.
- **Slow suite creep** — accepting "just one" integration-style test in
  the unit suite repeatedly until the suite takes minutes instead of
  seconds, and TDD's fast feedback loop is lost.
- **Test interdependence** — tests that only pass when run in a specific
  order because they share mutable state. Each test must set up and tear
  down its own state.
- **Vanity coverage** — writing tests to hit a coverage number without
  asserting anything meaningful (e.g., calling a function with no
  assertions at all). `coverage_analyzer.py` reports *line/branch*
  coverage, not assertion quality — pair it with `metrics_calculator.py`'s
  test quality scoring to catch this.

## 4. Triangulation

Triangulation is the technique of using multiple, increasingly general
examples to force an implementation from hard-coded to general, one small
step at a time — useful when the "obvious" general solution isn't obvious,
or when you want to resist over-engineering too early.

**Example: implementing `add(a, b)`**

Step 1 — first test, hard-code it green:
```python
def test_add_two_numbers():
    assert add(2, 3) == 5
```
```python
def add(a, b):
    return 5  # fakes it — simplest thing that passes
```

Step 2 — a second example that the fake can't satisfy forces
generalization:
```python
def test_add_different_numbers():
    assert add(10, 1) == 11
```
Returning a constant no longer works for both tests, so the implementation
is triangulated toward the real rule:
```python
def add(a, b):
    return a + b
```

Step 3 — additional examples (negative numbers, zero, floats) each act as
a check that the generalization actually holds, rather than a reason to
generalize further; if they all pass unchanged, the implementation is
already general enough.

**When to use triangulation:**
- The general rule isn't obvious from the first example, and faking a
  constant is genuinely the simplest first step.
- You want evidence-driven confidence in generality rather than
  assumption-driven confidence.

**When to skip it:** if the general implementation is just as simple as
the fake (as with `add` in practice), write the real implementation
immediately — triangulation is a tool for cases where jumping straight to
the general solution is uncertain or overly complex, not a mandatory ritual
for every test.

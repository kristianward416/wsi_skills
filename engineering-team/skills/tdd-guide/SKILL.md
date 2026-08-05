---
name: tdd-guide
description: Guides test-driven development using the red-green-refactor cycle. Use when starting TDD on a new feature, generating tests from requirements/user stories/API specs, detecting a project's language and test framework, scaffolding fixtures and mock data, adapting tests to Jest/Vitest/Pytest/JUnit/Mocha, analyzing coverage gaps from LCOV/JSON/XML reports, calculating test/code quality metrics, or producing red-green-refactor guidance and reports.
license: MIT
metadata:
  category: engineering-team
---

# TDD Guide

## Overview

TDD Guide is a workflow skill that drives disciplined test-driven development.
It does not just generate tests — it walks a developer or agent through the
**red-green-refactor** cycle step by step, detecting the project's stack,
scaffolding tests and fixtures, adapting output to the right test framework,
and verifying progress with coverage and quality metrics before declaring a
cycle done.

The skill is implemented as eight composable Python scripts under
`scripts/`. Each script does one job well and can be invoked independently,
but they are designed to be chained together in the order described below.

## When to use this skill

Use `tdd-guide` when:

- Starting TDD on a new feature or bug fix and you want red-green-refactor
  guidance rather than freeform test writing.
- Generating test cases from requirements, user stories, API specs, or
  existing code that has no tests yet.
- You don't know (or want to confirm) the language/framework in a repo
  before writing tests.
- You need realistic fixtures, mock objects, or test data.
- Translating generated/generic test logic into idiomatic Jest, Vitest,
  Pytest, JUnit, or Mocha code.
- Analyzing a coverage report (LCOV/JSON/XML) to find untested lines,
  branches, or files.
- Calculating test quality or code complexity metrics.
- Producing a concise, environment-appropriate report of TDD progress
  (desktop, CLI, or API context).

Do not use this skill for one-off exploratory scripts, or when the user
explicitly wants tests written after the implementation with no red-green
discipline (test-after) — in that case, at most `test_generator.py` and
`framework_adapter.py` are relevant, and the anti-pattern tradeoffs in
`references/tdd-best-practices.md` should be surfaced.

## The red-green-refactor workflow

TDD proceeds in short cycles. Each cycle has three phases, tracked by
`tdd_workflow.py`'s `TDDPhase` enum (`RED`, `GREEN`, `REFACTOR`) and its
`WorkflowState`:

1. **RED — write a failing test.**
   Before writing any test, run `format_detector.py` against the project
   root. It identifies the programming language, the testing framework
   already in use (or the best default if none), and relevant file
   conventions (e.g. `*.test.ts` vs `test_*.py`). This detection result is
   passed forward so every later step matches the project's actual stack
   instead of guessing.

   With the stack known, use `test_generator.py` to produce a first test
   case from the requirement, user story, or API spec under discussion.
   Where the test needs realistic input data, mocks, or edge-case values,
   call `fixture_generator.py` (pass a fixed seed when you need
   reproducible fixtures across runs). Run `tdd_workflow.py` to mark the
   cycle as entering `RED` and confirm the new test fails for the expected
   reason (missing implementation, not a typo or import error).

2. **GREEN — make it pass with the simplest code.**
   `tdd_workflow.py`'s `WorkflowState` advances to `GREEN` once the
   implementation change is made. If the generated test's syntax or
   idioms don't yet match the project's framework (for example
   `test_generator.py` emitted generic assertions), run
   `framework_adapter.py` to rewrite it into idiomatic Jest, Vitest,
   Pytest, JUnit, or Mocha — correct imports, describe/it or class/method
   structure, and framework-specific assertion/mocking calls. Re-run the
   test suite; `tdd_workflow.py` records the transition to green only when
   the target test (and the existing suite) passes.

3. **REFACTOR — improve structure without changing behavior.**
   With the suite green, refactor implementation or test code for clarity.
   After refactoring, re-run the full suite (still green) and use
   `coverage_analyzer.py` to parse the resulting coverage report (LCOV,
   JSON, or XML — see `assets/sample_coverage_report.lcov` for an example
   input) and confirm no regression in covered lines/branches, and to
   surface any gaps opened by the refactor. Use `metrics_calculator.py` to
   compute complexity and test-quality scores so the refactor is judged
   against real numbers, not intuition. `tdd_workflow.py` closes the cycle
   and is ready to start the next RED phase.

Throughout, `output_formatter.py` renders the current phase, test results,
coverage delta, and metrics as a context-appropriate report — compact for
CLI output, richer for desktop/chat contexts, structured JSON for API
consumers — using progressive disclosure so verbose detail is available
without flooding the default view.

## Script pipeline summary

| Step | Script | Purpose |
|---|---|---|
| 1 | `format_detector.py` | Detect language, framework, file conventions |
| 2 | `tdd_workflow.py` | Track RED → GREEN → REFACTOR phase and state |
| 3 | `test_generator.py` | Generate test cases from requirements/specs/code |
| 3 | `fixture_generator.py` | Generate fixtures, mock data, test data (seedable) |
| 4 | `framework_adapter.py` | Adapt tests to Jest/Vitest/Pytest/JUnit/Mocha idioms |
| 5 | `coverage_analyzer.py` | Parse LCOV/JSON/XML coverage, find gaps |
| 5 | `metrics_calculator.py` | Complexity, test quality, execution metrics |
| 6 | `output_formatter.py` | Context-aware reporting (Desktop/CLI/API) |

## Verification / definition of done

A TDD cycle is not complete until all of the following hold:

- [ ] The new test failed for the *correct* reason before implementation
      existed (confirmed RED, not a broken test).
- [ ] The full test suite passes after the implementation change (GREEN).
- [ ] The test is written in the idiomatic style of the detected framework
      (verified via `framework_adapter.py`, not left in generic form).
- [ ] Coverage did not regress versus the previous cycle, per
      `coverage_analyzer.py` (no accidental untested branches introduced).
- [ ] `metrics_calculator.py` shows no unexplained spike in complexity or
      drop in test quality score for the touched files.
- [ ] The behavior under test is asserted, not the implementation detail —
      see the anti-patterns in `references/tdd-best-practices.md`.
- [ ] A report was produced via `output_formatter.py` summarizing the
      phase, results, coverage, and metrics for this cycle.

If any box is unchecked, the cycle is not done — return to the relevant
phase rather than starting a new feature.

## References

- `references/tdd-best-practices.md` — Red-green-refactor discipline,
  test-first vs test-after tradeoffs, common anti-patterns, and the
  triangulation technique.
- `references/framework-guide.md` — Concrete per-framework conventions
  (Jest, Vitest, Pytest, JUnit, Mocha) with real code examples for what
  `framework_adapter.py` should produce.
- `references/ci-integration.md` — Wiring coverage gates and TDD checks
  into CI, including a working GitHub Actions example and flaky-test
  retry policy.

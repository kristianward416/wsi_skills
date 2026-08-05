# CI Integration

This reference covers wiring the TDD workflow into continuous integration:
turning coverage into a real merge gate (not just a dashboard number), a
working GitHub Actions pipeline that runs the test suite and feeds results
through `coverage_analyzer.py`, and a policy for handling flaky tests
without eroding trust in the "red means broken" signal TDD depends on.

## 1. Coverage thresholds as a merge gate

A coverage number that nobody enforces is trivia. To function as a gate it
must be:

1. **Computed on every PR**, not just on a schedule.
2. **Compared against a threshold that fails the build**, not merely
   reported.
3. **Scoped sensibly** — global thresholds catch long-term drift; diff
   coverage (coverage of *changed* lines only) catches regressions in the
   PR at hand without punishing a PR for pre-existing gaps elsewhere.

### Recommended threshold strategy

| Gate | Scope | Typical threshold | Enforced by |
|---|---|---|---|
| Global line coverage | Whole repo | Project-specific baseline (e.g. 80%), ratchet up over time, never down | `coverage_analyzer.py --mode=global --min-line=80` |
| Global branch coverage | Whole repo | Usually a few points below line coverage | `coverage_analyzer.py --mode=global --min-branch=70` |
| Diff coverage | Changed lines in the PR | Higher than global (e.g. 90-100%) — new code should be well tested even if legacy code isn't | `coverage_analyzer.py --mode=diff --min-line=90` |
| No new uncovered files | New files added in the PR | 100% of new files must have a corresponding test file | Custom check in CI script |

**Ratcheting, not fixed thresholds:** hard-coding "80%" forever means
coverage can coast at 80% indefinitely. A ratchet stores the last known
coverage percentage (e.g. in a committed `coverage-baseline.json`) and
fails the build if a PR's coverage is *lower* than the baseline, then
updates the baseline on merge to main. This makes the gate self-tightening
without needing to renegotiate a fixed number every quarter.

```json
// coverage-baseline.json
{ "line": 84.2, "branch": 71.5, "updated_at": "2026-08-01" }
```

## 2. GitHub Actions example

The pipeline below runs the test suite with coverage collection, feeds the
resulting report through `coverage_analyzer.py`, enforces both a global
ratchet and a diff-coverage gate on PRs, and uploads the human-readable
report as a build artifact.

```yaml
# .github/workflows/tdd-ci.yml
name: TDD CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test-and-coverage:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history so diff coverage can find the merge base

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov

      - name: Run tests with coverage
        run: |
          pytest \
            --cov=src \
            --cov-report=xml:coverage.xml \
            --cov-report=lcov:coverage.lcov \
            --junitxml=test-results.xml
        # Retry policy for flaky tests is handled inside pytest via
        # pytest-rerunfailures — see section 3, not by re-running this step.

      - name: Analyze coverage
        id: coverage
        run: |
          python scripts/coverage_analyzer.py \
            --input coverage.lcov \
            --format lcov \
            --baseline coverage-baseline.json \
            --min-line 80 \
            --min-branch 70 \
            --output coverage-report.json \
            --fail-under-baseline

      - name: Diff coverage gate (PRs only)
        if: github.event_name == 'pull_request'
        run: |
          python scripts/coverage_analyzer.py \
            --input coverage.lcov \
            --format lcov \
            --mode diff \
            --base origin/${{ github.base_ref }} \
            --min-line 90 \
            --fail-on-gap

      - name: Calculate test/code metrics
        run: |
          python scripts/metrics_calculator.py \
            --coverage coverage-report.json \
            --junit test-results.xml \
            --output metrics-report.json

      - name: Render report
        if: always()
        run: |
          python scripts/output_formatter.py \
            --context ci \
            --coverage coverage-report.json \
            --metrics metrics-report.json \
            >> "$GITHUB_STEP_SUMMARY"

      - name: Upload coverage artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-and-metrics
          path: |
            coverage.xml
            coverage.lcov
            coverage-report.json
            metrics-report.json
            test-results.xml
          retention-days: 14

      - name: Update coverage baseline (main only)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          cp coverage-report.json coverage-baseline.json
          git config user.name "ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git add coverage-baseline.json
          git commit -m "chore: update coverage baseline [skip ci]" || echo "no change"
          git push
```

Key points this pipeline demonstrates:

- **`fetch-depth: 0`** — diff coverage needs the merge base commit to know
  which lines actually changed; a shallow clone breaks that comparison.
- **Failure happens inside the analyzer step**, not just in a separate
  reporting step — `--fail-under-baseline` and `--fail-on-gap` cause
  `coverage_analyzer.py` to exit non-zero, which fails the job and blocks
  merge (when branch protection requires this check).
- **The baseline only updates on `main`**, never on a PR branch — otherwise
  a PR could silently lower the bar for everyone by merging a weaker
  baseline.
- **`$GITHUB_STEP_SUMMARY`** gets the human-readable report from
  `output_formatter.py` in CI context, visible directly on the Actions run
  page without opening artifacts.

### Branch protection

To make this an actual gate rather than an informational check, the repo
must require the `test-and-coverage` job (or a specific step's status) as
a required status check in branch protection settings for `main`. Without
that configuration, a red job is advisory only.

## 3. Flaky-test retry policy

Flaky tests — tests that pass or fail nondeterministically without a
related code change — are corrosive to TDD's core signal: red means
broken, green means working. A retry policy exists to keep CI usable while
that signal is repaired, not to hide flakiness permanently.

### Policy

1. **Retry sparingly and visibly, never silently.** Use a bounded retry
   (e.g. 2 reruns of a failed test) so a genuinely flaky test doesn't block
   unrelated merges, but every retry is logged and counted.

   ```
   pytest --reruns 2 --reruns-delay 1 --junitxml=test-results.xml
   ```

   ```javascript
   // jest.config.js
   module.exports = {
     testRetries: 2, // or via jest-circus retry API
   };
   ```

2. **A test that needed a retry to pass is not a clean pass.** Track
   rerun counts (pytest-rerunfailures and Jest both emit this in their
   JUnit output) and feed them into `metrics_calculator.py`'s test quality
   score — a test with a nonzero retry rate over the last N runs should
   visibly lower that file's quality score, not disappear once green.

3. **Quarantine, don't ignore.** If a test fails intermittently across
   multiple unrelated PRs, move it to a quarantine suite (e.g. tagged
   `@flaky` / marked `pytest.mark.flaky`) that runs but does not block
   merge, and open a tracked issue with the failure history attached. A
   quarantined test still runs on every CI build — it does not gate merges
   until it's fixed and un-quarantined — because a test that's simply
   deleted stops providing any signal at all.

   ```python
   @pytest.mark.flaky(reruns=0)  # quarantined: known intermittent, tracked in ISSUE-4821
   def test_eventually_consistent_cache_read():
       ...
   ```

4. **Set a quarantine expiry.** A quarantined test with no owner and no
   deadline becomes permanent dead weight. Require a linked issue and a
   review date (e.g. 2 sprints out) after which the test is either fixed
   or deleted — never left quarantined indefinitely by default.

5. **Root-cause before increasing retry counts.** Increasing
   `--reruns` from 2 to 5 to "make CI green" is a policy failure, not a
   fix — it usually means the underlying flakiness is a shared-state,
   timing, or ordering bug that will eventually cause a real production
   defect. The retry budget should stay fixed; flaky tests found are
   fixed or quarantined, not accommodated with more retries.

6. **Report flake rate, not just pass/fail.** `output_formatter.py`'s CI
   report should include a "tests that required a retry" section sourced
   from the JUnit rerun data, so flakiness trends are visible on every
   build summary rather than buried in raw logs.

### Example: pytest configuration

```ini
# pytest.ini
[pytest]
markers =
    flaky: known-intermittent test, quarantined and tracked (see linked issue)
addopts = --reruns 2 --reruns-delay 1 -m "not flaky or flaky"
```

```yaml
# separate, non-blocking job for quarantined tests
  quarantined-tests:
    runs-on: ubuntu-latest
    continue-on-error: true   # never blocks merge
    steps:
      - uses: actions/checkout@v4
      - run: pytest -m flaky --junitxml=quarantine-results.xml
      - uses: actions/upload-artifact@v4
        with:
          name: quarantine-results
          path: quarantine-results.xml
```

This keeps quarantined tests executing and visible (so regressions are
still caught and the flake is still being measured) while guaranteeing
they cannot block a merge that has nothing to do with their flakiness.

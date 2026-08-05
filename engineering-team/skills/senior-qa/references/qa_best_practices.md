# QA Best Practices

Process and quality-culture practices that sit around the test code itself: when to test, how to triage what testing finds, how to manage the data tests run against, and what actually blocks a merge. Use this reference when improving team-level test quality rather than an individual test file (see `testing_strategies.md` for strategy and `test_automation_patterns.md` for code-level patterns).

---

## 1. Shift-Left Testing

"Shift left" means moving quality activities earlier in the development timeline — toward design and coding — rather than treating testing as a phase that happens after the code is "done."

### Why it matters

The cost of fixing a defect grows roughly an order of magnitude at each stage it survives: catching a bug in code review costs minutes; catching it in QA costs hours; catching it in production costs incident response, customer trust, and possibly a hotfix deploy. Shift-left is the practice of pulling that detection point as far left as possible.

### Concrete shift-left practices for a React/Next.js team

1. **Static analysis in the editor, not just CI.** TypeScript strict mode and ESLint should catch type mismatches and unsafe patterns before a PR is opened, not after CI runs.
   ```jsonc
   // tsconfig.json
   { "compilerOptions": { "strict": true, "noUncheckedIndexedAccess": true } }
   ```
2. **Test-first for risky logic.** Write the failing test for a pricing calculation or auth guard *before* the implementation (classic TDD red-green-refactor) — it forces the acceptance criteria to be explicit before code exists.
3. **Acceptance criteria written as testable statements** at ticket creation, not after a feature is built. "Given a logged-out user visits /dashboard, then they are redirected to /login" is directly translatable into a Playwright test — a vague "handle auth properly" is not.
4. **QA involved in design review**, not just after a PR is opened — a QA engineer reviewing a Figma flow or API contract before implementation starts routinely catches missing error states and edge cases that are expensive to retrofit.
5. **Pre-commit hooks running fast checks** (lint, type-check, affected unit tests) so feedback arrives in seconds, not after a 10-minute CI run.
   ```bash
   # .husky/pre-commit
   npx lint-staged
   npm test -- --findRelatedTests --bail
   ```
6. **Contract tests (see `testing_strategies.md`) run against API changes before the frontend team builds against them**, catching integration mismatches at design time instead of integration time.

Shift-left does not mean "skip QA" — it means QA's involvement starts at requirements/design, and automated feedback loops run continuously rather than only in a dedicated end-of-cycle test phase.

---

## 2. Bug Triage and Severity Criteria

Every bug report needs two independent classifications: **severity** (technical impact) and **priority** (urgency to fix). Conflating them causes cosmetic-but-urgent bugs to get mis-scheduled and severe-but-rare bugs to get ignored.

### Severity levels

| Severity | Definition | Example |
|---|---|---|
| **S1 – Critical** | Data loss, security breach, or the application is unusable for a large user segment | Checkout charges the wrong amount; auth bypass; production outage |
| **S2 – High** | Major feature broken, no reasonable workaround | Search returns zero results for all valid queries |
| **S3 – Medium** | Feature partially broken, workaround exists | Filter dropdown doesn't reset on navigation, but reload fixes it |
| **S4 – Low** | Cosmetic, minor UX annoyance, no functional impact | Button misaligned by 4px on a rarely-visited settings tab |

### Priority levels (independent axis)

| Priority | Definition |
|---|---|
| **P0** | Fix immediately, block release / hotfix now |
| **P1** | Fix in the current sprint |
| **P2** | Fix in a future sprint, backlog with a target |
| **P3** | Fix opportunistically / won't-fix candidate |

A bug can be S1/P0 (production data corruption — drop everything) or S1/P2 (a critical bug in a feature flag that's off for 100% of users, so no urgency yet). Severity is about impact *if it occurs*; priority is about *when to act*, factoring in likelihood, exposure, and business context.

### Triage workflow

1. **Intake**: bug filed with repro steps, expected vs. actual, environment, and — if from exploratory testing — the charter that found it.
2. **Reproduce**: triage owner confirms repro before assigning severity; "cannot reproduce" bugs get a 2-week watch tag, not an immediate close.
3. **Classify**: assign severity using the table above, independent of who reported it or how loudly.
4. **Prioritize**: in a triage meeting (or async in the tracker), assign priority using severity + user impact (how many users/how often) + business context (is there a launch this week?).
5. **Regression-test on fix**: every fixed bug gets a corresponding automated test added before the fix PR merges — this is what prevents the same bug from reappearing silently (ties into the coverage-gap workflow via `scripts/coverage_analyzer.py --critical-paths`).
6. **Root-cause note on S1/S2**: a one-paragraph "why did this reach production" note, feeding back into shift-left practices (was there a missing test category? a missing design review?).

---

## 3. Test Data Management

Poorly managed test data is one of the largest hidden sources of flaky and slow test suites (see the isolation and non-determinism causes in `test_automation_patterns.md`).

### Principles

1. **Generate, don't hardcode, when uniqueness matters.**
   ```typescript
   import { faker } from '@faker-js/faker';
   const testUser = { email: faker.internet.email(), name: faker.person.fullName() };
   ```
2. **Use factories/builders for consistent, valid objects across tests**, so a schema change only requires updating one factory instead of every test file.
   ```typescript
   // test/factories/order.ts
   export function buildOrder(overrides: Partial<Order> = {}): Order {
     return {
       id: faker.string.uuid(),
       status: 'pending',
       items: [{ sku: 'sku-1', qty: 1, price: 10 }],
       total: 10,
       ...overrides,
     };
   }
   ```
3. **Seed deterministic fixtures for integration/E2E environments**, separate from production-like data, reset between runs:
   ```bash
   npm run db:seed:test   # runs before each CI E2E job, resets to a known baseline
   ```
4. **Never test against production data.** Use anonymized/synthetic data in staging; PII in test environments is both a compliance risk and a source of tests that "randomly" fail when a real customer's data changes.
5. **Isolate data per test, not per suite.** Prefer creating exactly the records a test needs inside that test (or a scoped `beforeEach`) over relying on a large shared seed dataset that every test silently depends on — shared fixtures are the #1 cause of order-dependent test failures.
6. **Clean up after E2E/integration runs that mutate a real backend**, either via transactional rollback (wrap the test in a DB transaction that's rolled back) or explicit teardown:
   ```typescript
   afterEach(async () => {
     await db.query('DELETE FROM orders WHERE test_marker = $1', [testRunId]);
   });
   ```
7. **Version fixture schemas alongside API/DB schema changes** — a fixture file that still reflects last quarter's API shape is a silent source of tests that pass against mocks but fail against reality (this is exactly the gap contract testing closes, per `testing_strategies.md`).

---

## 4. CI Gating Criteria: What Blocks a Merge vs. What's Advisory

Not every quality signal should block a PR — over-blocking trains engineers to route around CI (force-merge, disable checks), which is worse than having no gate at all. Split checks explicitly into **blocking** and **advisory**.

### Blocking (must pass to merge)

| Check | Rationale |
|---|---|
| Type check (`tsc --noEmit`) | Type errors are unambiguous defects, zero false-positive rate |
| Lint (errors, not warnings) | Catches real bugs (unused vars hiding a typo, missing deps in `useEffect`) |
| Unit + integration tests (`npm test -- --coverage`) | Core correctness signal; must be green |
| Coverage threshold on changed files (diff coverage, not just global) | Prevents coverage erosion PR-by-PR even if global % looks fine |
| E2E smoke suite (critical paths only: login, checkout, core CRUD) | High-confidence signal on the paths that matter most, kept small so it's fast and non-flaky enough to gate on |
| Security scan (dependency audit, secret scanning) | Non-negotiable regardless of feature intent |

```yaml
# .github/workflows/ci.yml (excerpt) — blocking checks as required status checks
jobs:
  test:
    steps:
      - run: npx tsc --noEmit
      - run: npm run lint -- --max-warnings=0
      - run: npm test -- --coverage
      - run: python scripts/coverage_analyzer.py coverage/ --threshold 80 --strict
      - run: npx playwright test --grep @smoke
```

### Advisory (surfaced, reviewed, does not block)

| Check | Rationale it's advisory, not blocking |
|---|---|
| Full E2E regression suite (beyond smoke) | Valuable but slower/flakier at full breadth; block release readiness, not every individual merge |
| Visual regression diffs (Chromatic/Percy) | Intentional UI changes *always* produce a diff — blocking would make every UI PR require a manual override, defeating the gate's signal value. Route to human review instead. |
| Bundle size delta | Informational unless a hard budget is breached; a small increase for a needed feature shouldn't hard-block |
| Lighthouse/performance score | Environment-noisy (CI runner variance); track trend, don't gate a single run |
| Mutation testing score (e.g., Stryker) | Useful test-quality signal but expensive to run on every PR; run nightly/weekly, review trend |
| Lint warnings (non-error rules, e.g., stylistic) | Style nits shouldn't block; fix opportunistically or via a periodic cleanup PR |

### Deciding whether a new check should block

Ask three questions:
1. **Is the false-positive rate near zero?** If a check can fail for reasons unrelated to a real defect (flaky E2E, visual diff on intentional redesign), it can't safely block — false-positive blocks teach engineers to ignore the gate.
2. **Is it fast enough to not become the bottleneck?** A blocking check that takes 20 minutes on every PR gets bypassed under deadline pressure; keep blocking suites fast (smoke-scope E2E, not full regression) and push comprehensive coverage to a nightly/merge-queue job.
3. **Is a human review step meaningfully better here than an automated pass/fail?** Visual and UX-adjacent signals are usually better as a required *reviewed* artifact (someone must look at the Chromatic diff and approve) than a binary gate.

### Escalation path for blocked PRs

- A legitimately flaky blocking check (confirmed via the flaky-test triage protocol in `test_automation_patterns.md`) can be re-run once; a second flake on the same PR should trigger quarantining the test, not repeated manual re-runs as standard practice.
- Emergency hotfixes may skip the full advisory review cycle but never skip blocking checks — if a hotfix needs to bypass type-check or the smoke suite, that's a signal the incident response process itself needs a documented emergency-merge exception, not a norm of quietly disabling CI.

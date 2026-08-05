# Test Automation Patterns

Implementation patterns for writing maintainable, reliable test code — Jest/RTL unit and integration tests, and Playwright E2E tests. Use this reference when writing or reviewing test code (see `testing_strategies.md` for what to prioritize, and `qa_best_practices.md` for process around flaky-test handling).

---

## 1. Page Object Model vs. Screenplay Pattern

Both patterns exist to stop E2E test files from becoming a tangle of duplicated selectors and actions. They solve the same problem at different levels of abstraction.

### Page Object Model (POM)

One class per page (or component), encapsulating its locators and the actions available on it. Tests call methods; they never touch selectors directly.

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorBanner: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Log in' });
    this.errorBanner = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorBanner).toHaveText(message);
  }
}
```

```typescript
// e2e/login.spec.ts
import { test } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

test('shows error on bad credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('bad@example.com', 'wrong');
  await loginPage.expectError('Invalid email or password');
});
```

**Strengths:** simple mental model, easy for a team new to E2E to adopt, matches `--include-pom` output from `scripts/e2e_test_scaffolder.py` in this skill.
**Weakness at scale:** page objects tend to accumulate every possible action on a page (god objects), and cross-page workflows (e.g., "complete checkout across 4 pages") end up orchestrated awkwardly in the test itself, mixing "what the user is trying to do" with "which page has which button."

### Screenplay Pattern

Models tests around **actors** performing **tasks** built from composable **interactions**, decoupling the "what" (business intent) from the "how" (UI mechanics). Better suited to large suites with many cross-cutting workflows and multiple actor roles (e.g., admin vs. customer).

```typescript
// e2e/screenplay/actors.ts
import { Page } from '@playwright/test';

export class Actor {
  constructor(public page: Page) {}
  async attemptsTo(...tasks: Task[]) {
    for (const task of tasks) await task.performAs(this);
  }
}

export interface Task {
  performAs(actor: Actor): Promise<void>;
}
```

```typescript
// e2e/screenplay/tasks/LogIn.ts
export class LogIn implements Task {
  static withCredentials(email: string, password: string) {
    return new LogIn(email, password);
  }
  private constructor(private email: string, private password: string) {}

  async performAs(actor: Actor) {
    await actor.page.getByLabel('Email').fill(this.email);
    await actor.page.getByLabel('Password').fill(this.password);
    await actor.page.getByRole('button', { name: 'Log in' }).click();
  }
}
```

```typescript
// e2e/checkout.spec.ts
test('customer completes checkout', async ({ page }) => {
  const customer = new Actor(page);
  await customer.attemptsTo(
    LogIn.withCredentials('jane@example.com', 'secret'),
    AddToCart.item('sku-123'),
    Checkout.withSavedCard(),
  );
  await expect(page.getByText('Order confirmed')).toBeVisible();
});
```

**Strengths:** reads like business language, tasks compose across pages, scales to multi-role/multi-workflow suites.
**Weakness:** more upfront ceremony/abstraction; overkill for a small app with 5 pages and one user role.

### Choosing

| Situation | Pattern |
|---|---|
| Small-to-mid app, single user role, scaffolded via `e2e_test_scaffolder.py --include-pom` | Page Object Model |
| Large suite, multiple actor roles, workflows spanning many pages | Screenplay |
| Team unfamiliar with test architecture patterns | Start with POM, migrate pieces to Screenplay if pain shows up |

---

## 2. Flaky Test Root Causes and Fixes

A flaky test passes and fails intermittently with no code change. Flaky tests erode trust in the suite — teams start ignoring failures ("just rerun it"), which defeats the point of having tests. Root-cause by category:

### A. Improper waits / race conditions

**Symptom:** test fails only under load, in CI, or "sometimes."

```typescript
// BAD: fixed sleep — either too short (flaky) or too long (slow)
await page.click('button');
await page.waitForTimeout(1000);
expect(await page.textContent('.result')).toBe('Done');

// GOOD: wait for the actual condition
await page.click('button');
await page.getByText('Done').waitFor({ state: 'visible' });
```

```typescript
// RTL: BAD — assumes the DOM has updated synchronously
fireEvent.click(button);
expect(screen.getByText(/success/i)).toBeInTheDocument();

// GOOD — findBy* retries until the element appears or times out
fireEvent.click(button);
expect(await screen.findByText(/success/i)).toBeInTheDocument();
```

Rule: never use `waitForTimeout`/`sleep`. Always wait on a *signal* — an element appearing/disappearing, a network response (`page.waitForResponse`), or a URL change (`page.waitForURL`).

### B. Test isolation failures (shared/leaked state)

**Symptom:** test passes alone, fails in the full suite; order-dependent failures.

Common causes:
- Module-level mutable state not reset between tests (`let cache = {}` at module scope).
- MSW handlers added in one test leaking into the next (`server.resetHandlers()` missing from `afterEach`).
- Playwright tests sharing a logged-in browser `storageState` and one test logging out.
- Database/fixture data not cleaned up, so test B sees rows test A created.

```typescript
// Enforce reset every test
afterEach(() => {
  server.resetHandlers();
  jest.clearAllMocks();
  cleanup(); // RTL — usually automatic in modern RTL, but explicit is safer with custom setups
});
```

For Playwright, isolate state per test/worker rather than reusing a shared account:

```typescript
// playwright.config.ts
export default defineConfig({
  use: { storageState: undefined }, // don't default to a shared session
  fullyParallel: true,
});

// Per-test isolated auth via fixture, not a hardcoded shared account
test('checkout as fresh user', async ({ browser }) => {
  const context = await browser.newContext(); // isolated cookies/storage
  const page = await context.newPage();
  // ...
  await context.close();
});
```

### C. Non-deterministic or environment-dependent test data

**Symptom:** fails on specific dates, locales, or when run twice against the same DB.

```typescript
// BAD: depends on "today" — fails at year boundaries, different timezones
expect(formatDate(new Date())).toBe('Aug 5, 2026');

// GOOD: inject/freeze the clock
import { vi } from 'vitest'; // or jest.useFakeTimers()
beforeEach(() => vi.setSystemTime(new Date('2026-08-05T12:00:00Z')));
```

```typescript
// BAD: reuses a fixed email — second run fails with "user already exists"
const email = 'test@example.com';

// GOOD: generate unique, deterministic-per-run test data
import { faker } from '@faker-js/faker';
const email = `qa-${Date.now()}-${faker.string.alphanumeric(6)}@example.com`;
```

### D. Third-party/network dependency

**Symptom:** fails when a real external service is slow/down, or rate-limits CI.

Fix: mock at the network boundary for unit/integration tests (MSW), and for E2E either mock the third-party call (`page.route()`) or run against a stable staging double, never the live production third party.

```typescript
// Playwright: intercept and stub a flaky third-party widget
await page.route('**/analytics-widget.js', (route) => route.fulfill({ status: 200, body: '' }));
```

### Flaky test triage protocol

1. Quarantine, don't delete: tag with `test.fixme()` / `@flaky` and file a ticket — never silently skip without tracking.
2. Reproduce locally with `--repeat-each=20` (Playwright) or looped `jest --runInBand` to confirm it's real flake vs. a one-off infra blip.
3. Fix at the root cause category above; re-run 20x before removing quarantine.
4. Track flake rate per test in CI dashboards; a test flaking >2% of runs over 2 weeks gets rewritten or removed.

---

## 3. Parallel Execution and Sharding Strategies

### Jest parallelization

Jest parallelizes across test **files** (not within a file) using worker processes:

```bash
npm test -- --maxWorkers=4        # cap workers (CI containers often have limited CPU)
npm test -- --maxWorkers=50%      # use half of available cores
```

Isolation requirement: because files run in separate workers/processes, any shared external resource (a real DB, a fixed port) must be workerized or the parallel runs collide. Prefer mocks (MSW) over real network for exactly this reason.

### Playwright parallelization and sharding

Playwright parallelizes at two levels: **workers** within one CI machine, and **shards** across multiple CI machines.

```typescript
// playwright.config.ts
export default defineConfig({
  fullyParallel: true,
  workers: process.env.CI ? 4 : undefined,
});
```

```yaml
# .github/workflows/e2e.yml — shard across 4 machines
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npx playwright test --shard=${{ matrix.shard }}/4
  - uses: actions/upload-artifact@v3
    with:
      name: blob-report-${{ matrix.shard }}
      path: blob-report/
```

```bash
# Merge sharded reports after all shards complete
npx playwright merge-reports --reporter=html ./all-blob-reports
```

Sharding rule of thumb: shard when total E2E suite runtime exceeds ~10 minutes on one machine, or when CI minutes cost more than the marginal machine cost of running in parallel. Balance shards by test *duration*, not test *count* — Playwright's built-in sharding does this automatically via its test-duration cache once `--shard` has been run a few times.

### Cross-worker isolation checklist

- Each Playwright worker gets its own browser context — never share `page`/`context` objects across `test()` blocks via module-level variables.
- Test data must be unique per worker (`test.info().parallelIndex` can seed unique IDs).
- Avoid asserting on global counts ("there are exactly 3 users") when tests run in parallel against a shared backend — assert on the specific entity you created.

---

## 4. Visual Regression Testing

Functional tests (does the click work) don't catch CSS/layout regressions (the button is now off-screen, text overflows a container). Visual regression testing screenshots the rendered UI and diffs it pixel-by-pixel (or via perceptual diff) against an approved baseline.

### Playwright's built-in screenshot assertions

```typescript
// e2e/visual/dashboard.spec.ts
import { test, expect } from '@playwright/test';

test('dashboard visual regression', async ({ page }) => {
  await page.goto('/dashboard');
  await page.waitForLoadState('networkidle');
  // Mask dynamic content that legitimately changes (timestamps, avatars)
  await expect(page).toHaveScreenshot('dashboard.png', {
    mask: [page.getByTestId('last-updated-timestamp')],
    maxDiffPixelRatio: 0.01,
  });
});
```

```bash
# First run creates the baseline; review and commit it
npx playwright test --update-snapshots
```

### Dedicated visual regression tools

For cross-browser/cross-viewport visual coverage at scale, teams typically graduate from Playwright's built-in snapshots to a hosted diffing service:

- **Chromatic** (Storybook-native): snapshots every Storybook story on each PR, flags visual diffs for human approval directly in the PR UI.
- **Percy**: framework-agnostic, integrates with Playwright/Cypress, handles cross-browser rendering farms so diffs aren't polluted by font-rendering differences between a dev's laptop and CI.
- **Applitools Eyes**: uses AI-based "visual AI" matching that tolerates anti-aliasing noise better than strict pixel diffing.

### Practical rules to avoid a noisy visual suite

1. **Mask or freeze non-deterministic regions** — clocks, avatars, ads, animations (`animations: 'disabled'` in Playwright's screenshot options).
2. **Pin fonts and disable animations globally** in the visual test config so CI rendering matches the approved baseline machine consistently.
3. **Review diffs as a required PR step**, not an auto-block — visual regression is best treated as advisory-with-human-review (see the advisory-vs-blocking distinction in `qa_best_practices.md`) because legitimate intentional UI changes will "fail" the diff every time.
4. **Scope to key pages/components**, not every route — visual regression suites become maintenance burdens fast if applied indiscriminately; target pages with high design-fidelity requirements (marketing pages, checkout) and reusable UI kit components (Storybook + Chromatic is the standard pairing here).

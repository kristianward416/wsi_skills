# Testing Strategies

Strategic frameworks for deciding *what* to test, *how much*, and *in what proportions* for React/Next.js applications. Use this reference when designing a test strategy before writing test code (see `test_automation_patterns.md` for implementation and `qa_best_practices.md` for process/quality gates).

---

## 1. Test Pyramid vs. Testing Trophy

### The Test Pyramid (Mike Cohn)

The classic pyramid optimizes for speed and isolation by weighting effort toward the bottom:

```
        /\
       /E2E\        <- few, slow, expensive, high confidence
      /------\
     /  Integ  \    <- more, moderate speed
    /------------\
   /   Unit Tests  \ <- many, fast, cheap, isolated
  /------------------\
```

- **Unit (70%)**: pure functions, hooks (`renderHook`), reducers, utility modules. No DOM, no network.
- **Integration (20%)**: multiple units composed — a form with validation, a component wired to context/state.
- **E2E (10%)**: full user journeys through a real browser (Playwright).

Rationale: unit tests are cheap to write and run in milliseconds, so you want the largest volume there; E2E tests are valuable but slow and flaky, so keep the count small and target only critical paths (checkout, auth, core CRUD).

**Where the pyramid breaks down for UI-heavy apps:** unit-testing React components in isolation (shallow rendering, testing internal state/props directly) tends to test implementation details, not user-facing behavior. This produces tests that pass while the UI is broken, or that break on harmless refactors — a classic false-positive/false-negative trap.

### The Testing Trophy (Kent C. Dodds)

For React/Next.js apps, prefer the **testing trophy**, which shifts weight to integration tests:

```
      /‾‾‾‾‾‾‾‾\
     /   E2E    \      <- few
    /------------\
   / Integration  \    <- most (the "trophy cup")
  /----------------\
 /   Unit           \  <- some
/______________________\
     Static (types/lint)
```

- **Static analysis (base)**: TypeScript, ESLint — catches typos, wrong prop types, unused variables before a test even runs. Free confidence.
- **Unit**: reserved for complex, isolated logic — pricing calculations, date utilities, reducers, custom hooks with `renderHook`.
- **Integration (the bulk)**: render a component (or a small tree of components) with React Testing Library the way a user would interact with it — click, type, assert on visible text/roles. This is where "render with real providers, mock only the network via MSW" tests live. They give the best confidence-to-cost ratio because they exercise real component composition without the cost of a full browser.
- **E2E**: reserved for the handful of flows where cross-system integration risk is highest (payment, auth redirect, multi-page wizards).

**Decision rule for this repo's `scripts/test_suite_generator.py` output:** generated component test stubs should default to trophy-style integration tests (render + `screen.getByRole` + `fireEvent`/`userEvent`), not shallow/implementation-detail unit tests. Reserve true unit tests for `src/lib/`, `src/utils/`, and custom hooks.

### Choosing between them

| Signal | Lean Pyramid | Lean Trophy |
|---|---|---|
| Heavy business logic, few UI states | Yes | |
| Component-driven UI, many visual states | | Yes |
| Team new to testing, wants fast wins | | Yes (integration tests catch more per test) |
| Library/SDK code | Yes | |

---

## 2. Risk-Based Test Prioritization

Not all code deserves equal test investment. Prioritize using a risk score combining **likelihood of failure** and **impact of failure**.

### Risk matrix

```
              Low Impact        High Impact
Low Likelihood   Skip/Smoke     Regression suite
High Likelihood  Unit + lint    Exhaustive (unit+integration+E2E)
```

Factors that raise likelihood:
- Recently changed code (churn) — correlate with git blame / commit frequency.
- Complex conditional logic (high cyclomatic complexity).
- Third-party/external integration points (payment gateways, auth providers).
- Code written by a new contributor or under deadline pressure.

Factors that raise impact:
- Revenue path (checkout, subscription billing).
- Data integrity (irreversible writes, deletes).
- Compliance/security (PII handling, auth/authorization).
- Blast radius (shared components used across 20+ pages vs. a one-off page).

### Applying it to coverage targets

Use risk tiers to set differentiated coverage thresholds instead of one flat number:

```javascript
// jest.config.js — tiered thresholds
module.exports = {
  coverageThreshold: {
    global: { branches: 70, functions: 70, lines: 70, statements: 70 },
    './src/lib/payments/**': { branches: 95, functions: 95, lines: 95, statements: 95 },
    './src/components/ui/**': { branches: 60, functions: 60, lines: 60, statements: 60 },
  },
};
```

Run `python scripts/coverage_analyzer.py coverage/ --critical-paths` to flag under-tested high-risk modules specifically, rather than chasing a uniform global percentage — 80% global coverage is meaningless if the uncovered 20% is the discount-code branch in checkout.

### Prioritization checklist when time is limited

1. What broke in production in the last 2 releases? Test that first (regression).
2. What has the highest revenue/legal impact if wrong?
3. What changed in this PR? (diff-based test targeting)
4. What has zero coverage today per the coverage analyzer's gap report?
5. Everything else, in order of user traffic (analytics-informed).

---

## 3. Exploratory Testing Charters

Scripted/automated tests verify *known* requirements. Exploratory testing finds the *unknown* — edge cases, usability issues, and integration surprises that no one thought to write a test for. It is time-boxed and structured with a **charter**, not ad-hoc clicking.

### Charter template (session-based test management, James Bach / Jon Bach)

```
Charter: Explore [area] with [resources] to discover [risk/information]
Time-box: 60–90 minutes
```

Example charters for a Next.js e-commerce app:

- "Explore the checkout flow with an interrupted network (throttled/offline) to discover state-consistency bugs."
- "Explore the signup form with clipboard paste, autofill, and IME (composed) input to discover validation gaps."
- "Explore the dashboard with a screen reader and keyboard-only navigation to discover a11y regressions."
- "Explore product search with boundary-value queries (empty, single char, 500-char, emoji, SQL-injection-like strings) to discover crash/XSS risk."

### Session structure

1. **Charter** (as above) — scope it narrowly enough to finish in one session.
2. **Setup** — pick test data, environment, tools (DevTools open, network throttling on).
3. **Test** — explore, deviate when something looks interesting ("follow your nose").
4. **Notes** — log as you go: what you tried, what you noticed, bugs found, questions raised. Use a simple format: `Time | Action | Observation | Bug? (Y/N)`.
5. **Debrief** — within the same day, convert notable findings into either a filed bug (see triage criteria in `qa_best_practices.md`) or a new automated regression test so the same bug can't slip through unnoticed next time.

### When to run exploratory sessions

- Before a major release, on top of (not instead of) the automated suite.
- After a large refactor, targeting the changed area specifically.
- When a bug report is vague ("sometimes checkout doesn't work") — exploratory testing is how you reproduce it before you can write a regression test for it.
- New feature acceptance, in parallel with the PR review, using the acceptance criteria as charter seeds.

Exploratory testing is not a replacement for the automated pyramid/trophy — it's the feedback loop that tells you *which* new automated tests to write next.

---

## 4. Contract Testing for Microservices

When a Next.js frontend (or a BFF layer) consumes multiple backend services, E2E tests that spin up every real service are slow, flaky, and expensive to maintain. **Contract testing** verifies that a consumer and provider agree on the shape of their interaction without either side needing the other running.

### Consumer-driven contracts (Pact)

The consumer (e.g., the Next.js app) defines the interactions it expects; the contract is published and the provider (API) verifies it can satisfy that exact contract in its own CI.

**Consumer side** (generates the contract):

```typescript
// pact/users-api.pact.test.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';

const provider = new PactV3({ consumer: 'web-app', provider: 'users-api' });

test('get user by id returns user', () => {
  provider
    .given('user 123 exists')
    .uponReceiving('a request for user 123')
    .withRequest({ method: 'GET', path: '/api/users/123' })
    .willRespondWith({
      status: 200,
      body: {
        id: MatchersV3.integer(123),
        name: MatchersV3.string('Jane Doe'),
        email: MatchersV3.email('jane@example.com'),
      },
    });

  return provider.executeTest(async (mockServer) => {
    const res = await fetch(`${mockServer.url}/api/users/123`);
    expect(res.status).toBe(200);
  });
});
```

**Provider side** (verifies the contract in the API's own CI, using the published pact file from a broker):

```javascript
// The provider CI pulls the contract from Pact Broker and replays each
// interaction against the real provider implementation to confirm it
// still returns a response matching the agreed shape.
```

### Why this matters more than mocking with MSW alone

MSW (already used in this skill's Jest tests) mocks the API *as the frontend team believes it behaves*. If the backend changes its response shape, MSW mocks silently continue to pass while the real integration breaks in production. Contract testing closes that gap because the *provider* verifies the same contract, so a breaking API change fails the provider's CI, not just a page in production.

### Schema-based alternative (OpenAPI/JSON Schema contract checks)

When a full Pact broker is overkill, a lighter-weight contract check validates real API responses (in a nightly or pre-deploy job) against the published OpenAPI schema:

```bash
# Validate live API responses against OpenAPI spec
npx openapi-response-validator --spec ./openapi.yaml --url https://api.staging.example.com
```

### Where contract tests sit in the strategy

They fill the gap between mocked integration tests (fast, but can drift from reality) and full E2E tests (accurate, but slow and require every service up). Treat them as a required CI gate for any PR that changes a shared API's request/response shape — see the CI gating criteria in `qa_best_practices.md`.

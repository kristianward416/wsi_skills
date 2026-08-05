# Frontend Best Practices

Baseline standards this skill enforces on every recommendation: measurable performance targets, an accessibility floor, a decision tree for state management, and a testing shape. These are the concrete numbers referenced by the "Verifiable success criteria" section of `SKILL.md` — use this doc to justify *why* those numbers, not just what they are.

## 1. Core Web Vitals — actual thresholds

Google's Core Web Vitals are measured at the **75th percentile of real users** (p75, field data via CrUX/RUM), not lab averages. A page that looks fast in one Lighthouse run can still fail CWV if the 75th-percentile real-world experience (slower device, worse network) is bad.

| Metric | What it measures | Good | Needs Improvement | Poor |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Time until the largest visible element (hero image, headline block) renders | ≤ 2.5s | 2.5s – 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | Responsiveness — delay between a user interaction and the next visual update | ≤ 200ms | 200ms – 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Sum of unexpected layout shift scores during the page's lifetime | ≤ 0.1 | 0.1 – 0.25 | > 0.25 |

INP replaced FID (First Input Delay) as a Core Web Vital in March 2024 — if you're referencing FID targets (≤100ms) in a legacy audit, they no longer count toward CWV pass/fail; re-baseline against INP.

**Per-profile LCP targets used by this skill's decision engine** (from `SKILL.md`, mobile-4G p75 unless noted):

| Profile | LCP target | Rationale |
|---|---|---|
| `astro-or-static` | 1200ms | Near-zero JS, content is mostly static HTML — no excuse for slow LCP |
| `remix-or-sveltekit` | 1500ms | Progressive enhancement, low-JS-first |
| `next-app-router` | 2000ms | RSC-first but still React hydration cost |
| `vite-spa` | 2500ms | Auth-walled, desktop-primary — SPA bootstrap cost is tolerated |

**Common CLS causes and fixes:**
- Images/iframes without explicit dimensions → always set `width`/`height` or use `aspect-ratio` in CSS.
- Web fonts swapping in and reflowing text → use `font-display: swap` with size-adjusted fallback fonts (or `next/font`'s automatic metric matching).
- Ads/embeds injected above existing content → reserve space with a fixed-height container before the content loads.
- Content shifting from an async data fetch replacing a skeleton of different height → make skeleton heights match final content exactly.

**Common INP causes and fixes:**
- Large synchronous work on the main thread during a click handler (e.g., re-rendering a huge list, running a heavy computation inline) → break work up with `startTransition` / `useTransition`, or move it off the main thread (Web Worker) for anything over ~50ms.
- Third-party scripts (analytics, chat widgets) blocking the main thread right when a user interacts → load with `next/script`'s `strategy="afterInteractive"` or `"lazyOnload"`, never blocking.

## 2. Accessibility Baseline (WCAG)

Default floor for this skill's recommendations: **WCAG 2.1 AA**, unless the intake questions (see `SKILL.md`'s forcing-question library, Q7) specify AAA or explicit "best-effort." AA is what most legal/compliance frameworks (ADA, Section 508, EN 301 549) actually reference.

**Non-negotiable AA checklist:**

1. **Color contrast** — minimum **4.5:1** for normal text, **3:1** for large text (≥18pt/24px or ≥14pt/19px bold). UI components and graphical objects need **3:1** against adjacent colors.
2. **Keyboard operability** — every interactive element reachable and operable via keyboard alone, in a logical tab order. No keyboard traps (a focus that enters a widget and can't `Tab`/`Escape` back out).
3. **Visible focus indicator** — never `outline: none` without a replacement. Use `:focus-visible` to style focus rings without showing them on mouse clicks:

```css
button:focus-visible {
  outline: 2px solid theme(colors.blue.500);
  outline-offset: 2px;
}
```

4. **Semantic HTML before ARIA** — a native `<button>` gets keyboard handling, focus, and role for free; a `<div onClick>` requires you to manually reimplement all of it (`role="button"`, `tabIndex={0}`, `onKeyDown` for Enter/Space). Reach for ARIA only when no native element covers the semantics (custom comboboxes, tab panels, live regions).
5. **Accessible names for icon-only controls:**

```tsx
<button type="button" aria-label="Close dialog" onClick={onClose}>
  <XIcon aria-hidden="true" />
</button>
```

6. **Skip links** for keyboard users to bypass repeated navigation:

```tsx
<a href="#main-content" className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4">
  Skip to main content
</a>
```

7. **Form labels** — every input has a programmatically associated label (`<label htmlFor>`, not just placeholder text — placeholders disappear on input and aren't reliably read by all screen readers).
8. **Live regions for async updates** — status messages that appear without a page navigation (form submission success/error, toast notifications) need `aria-live="polite"` (or `"assertive"` for errors) so screen reader users hear them without needing to find them visually.

**CI gate:** run `axe-core` (via `@axe-core/playwright` or `jest-axe`) on every component/page in CI, and treat any AA violation as a build failure, not a warning. Manual testing (keyboard-only pass, screen reader spot-check with VoiceOver/NVDA) should still happen before major releases — automated tools catch roughly 30-50% of real WCAG issues; they're a floor, not a substitute for manual review.

## 3. State Management Decision Tree

Most over-engineering in frontend codebases comes from reaching for global state before local state is exhausted. Work through these in order:

```
Does the state affect only one component's own render (a toggle, an input value,
a hover state)?
  → useState / useReducer, colocated in that component. STOP HERE for most state.

Does the state need to be shared by a few components that are close together
in the tree (a form and its fields, a tab group and its panels)?
  → Lift state to their lowest common ancestor. Still local useState/useReducer,
    just placed higher. See "State Colocation" in react_patterns.md.

Does the state need to be read by many components scattered across different
subtrees, but changes INFREQUENTLY (theme, locale, authenticated user, feature
flags)?
  → React Context. Split by concern (see "Context Performance Pitfalls" in
    react_patterns.md) and memoize the provider value.

Does the state change FREQUENTLY and is read by many components scattered
across different subtrees (shopping cart, real-time collaborative doc state,
a design tool's canvas state)?
  → External store with selector-based subscriptions: Zustand, Jotai, or
    Redux Toolkit. Context re-renders every consumer on every change; these
    libraries re-render only the components whose selected slice changed.

Is the state actually SERVER data (fetched from an API, needs caching,
revalidation, dedup, background refetch)?
  → Not client state at all. Use a server-cache library: React Query / TanStack
    Query, SWR, or (in RSC-first Next.js apps) fetch() with Next's built-in
    cache + revalidation. Do not put server data into useState/Redux/Context —
    you'll end up hand-rolling cache invalidation, loading states, and race
    condition handling that these libraries already solved.

Does the state need to persist across page reloads (draft form content,
user preferences)?
  → useState + a useLocalStorage/useSessionStorage hook, or URL search params
    (nuqs, useSearchParams) if the state should be shareable/bookmarkable
    (filters, pagination, active tab).
```

**Rule of thumb on Context specifically:** Context is for *dependency injection of rarely-changing values*, not a general state manager. If you find yourself debugging why an unrelated component re-rendered, and the answer is "it reads a Context whose provider re-renders often," that's the signal to move to an external store.

## 4. Testing Pyramid for Frontend

```
        ▲
       / \        E2E (Playwright / Cypress)
      /___\       ~5-10% of tests — critical user journeys only
     /     \      (signup, checkout, primary CRUD flow)
    /       \
   /_________\    Integration (Testing Library + component tree)
  /           \   ~20-30% — component + its real children, mocked network
 /             \  (a form submits, a modal opens and traps focus)
/_______________\ Unit (Vitest/Jest)
                  ~60-70% — pure functions, hooks in isolation,
                  reducers, utility logic
```

**Unit tests** — pure logic, no rendering. Fast (milliseconds), cheap to write, should cover the majority of business logic:

```ts
// lib/pricing.test.ts
import { calculateDiscount } from './pricing';

test('applies 10% discount for orders over $100', () => {
  expect(calculateDiscount(150)).toBe(15);
});
```

**Integration tests** — React Testing Library, rendering a real component tree, querying by accessible role (not implementation detail like class names), simulating real user interaction:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('submitting the form with valid input calls onSubmit with the values', async () => {
  const onSubmit = vi.fn();
  render(<SignupForm onSubmit={onSubmit} />);

  await userEvent.type(screen.getByLabelText('Email'), 'a@example.com');
  await userEvent.type(screen.getByLabelText('Password'), 'correct-horse');
  await userEvent.click(screen.getByRole('button', { name: 'Sign up' }));

  expect(onSubmit).toHaveBeenCalledWith({ email: 'a@example.com', password: 'correct-horse' });
});

test('dialog exposes correct accessibility semantics', () => {
  render(<Dialog open title="Confirm deletion" />);
  const dialog = screen.getByRole('dialog');
  expect(dialog).toHaveAttribute('aria-labelledby');
});
```

Guiding principle (Kent C. Dodds' Testing Library philosophy): **query the way a user or assistive technology would** — `getByRole`, `getByLabelText`, `getByText` — not `getByTestId` or querying CSS classes. A test that queries by role fails when the accessible experience breaks, which is exactly the failure you want caught; a test that queries by `data-testid` can pass while the component is unusable to a screen reader.

**E2E tests** — real browser, real network (or a realistic mock server), covering only the handful of flows where a regression is catastrophic (checkout, auth, primary conversion path). Expensive to run and maintain (seconds per test, flaky across environments), so keep this layer small and targeted — E2E should catch integration failures across systems (frontend + API + DB), not component-level bugs that unit/integration tests already cover cheaper and faster.

**Where visual regression fits:** not really part of the pyramid — it's an orthogonal layer (Chromatic, Playwright screenshot comparison) that catches unintended CSS/layout changes unit and integration tests can't see. Add it for design-system components and critical marketing pages, not every component.

**CI gate tied to `SKILL.md`'s verifiable success criteria:** every PR should run unit + integration tests plus an automated a11y check (axe-core) and a Lighthouse budget check (perf floor + a11y floor) against the profile's thresholds — a recommendation or PR that doesn't state its Lighthouse perf/a11y floor is incomplete, per the Karpathy discipline in `SKILL.md`.

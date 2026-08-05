# Development Workflows

How code moves from a developer's machine to production, and how a team stays coordinated while it does. This document assumes a stack and architecture have already been chosen (see `tech_stack_guide.md` and `architecture_patterns.md`) — it's about process, not structure.

Every recommendation here is calibrated against the four customization profiles and must ultimately support the three verifiable success criteria this skill requires on every recommendation: API latency (p50/p95/p99), frontend perf (LCP/INP/CLS), and uptime/SLO.

---

## 1. Git workflow: trunk-based vs. GitFlow

| | Trunk-based | GitFlow |
|---|---|---|
| **Branch model** | One long-lived branch (`main`); short-lived feature branches (hours to 2-3 days), merged behind flags if incomplete | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` — multiple long-lived branches |
| **Merge frequency** | Multiple times/day per engineer | Feature branches often live days-to-weeks before merging to `develop` |
| **Deploy cadence fit** | Matches `daily`/`per-PR` cadence (the `saas-startup` and `enterprise-scale` defaults) | Matches `weekly`/`quarterly` release-train cadence |
| **Incomplete features in `main`** | Hidden behind feature flags (§2), not behind a branch | Isolated on `feature/*` until "done," then merged |
| **Merge conflict cost** | Low — small diffs, frequent integration surfaces problems early | High — long-lived branches drift from `main`/`develop`, big-bang merges are conflict-prone |
| **Pick when** | Team practices continuous delivery, has CI gates and feature flags to de-risk incomplete work in `main` | Regulated release trains with scheduled, audited release windows; team isn't ready to invest in feature-flag infrastructure |

**Default for `saas-startup` and `enterprise-scale` profiles (per-PR/daily cadence): trunk-based.** *Accelerate* (Forsgren et al.) is explicit on this: trunk-based development with short-lived branches correlates with higher deployment frequency and lower change failure rate — the two together are what "daily cadence" in this skill's assumptions actually requires operationally. GitFlow's branch ceremony is a mismatch for a team that has committed to daily+ deploys; it reintroduces the long-lived-branch merge pain trunk-based is designed to eliminate.

**When GitFlow is still the right call:** regulated `enterprise-scale` contexts with externally-mandated release windows (e.g., quarterly certified releases in some financial/healthcare contexts) where a `release/*` branch is the audit artifact reviewers sign off on. This is a compliance-driven exception, not a default.

**Conventional commits, regardless of branch model:**

```
feat(billing): add proration for mid-cycle plan upgrades
fix(auth): rotate refresh token on every use, not just expiry
chore(deps): bump prisma to 5.20.0
docs(api): document rate-limit headers on /v1/orders
```

This isn't cosmetic — `feat`/`fix`/`chore` prefixes drive automated changelog generation and semantic-version bumps (`semantic-release`), which matters the moment you have external API consumers who read a changelog to know if a release is breaking.

---

## 2. Feature-flag-driven development

Feature flags are what makes trunk-based development safe for incomplete work: a half-finished feature merges to `main` behind a flag, ships to production dark, and gets enabled per-user/per-percentage without a deploy.

```
if (flags.isEnabled('new-checkout-flow', { userId })) {
  return <NewCheckoutFlow />;
} else {
  return <LegacyCheckoutFlow />;
}
```

| Flag type | Purpose | Lifespan |
|---|---|---|
| **Release flag** | Hide incomplete work in `main` until ready | Days-weeks, then deleted once fully rolled out |
| **Experiment flag** | A/B test a change against a metric | Weeks, deleted after the experiment concludes |
| **Ops flag** | Kill switch for a risky dependency/feature under load | Long-lived, rarely deleted (it's your incident-response lever) |
| **Permission flag** | Gate a feature by plan tier/entitlement | Long-lived, tied to business logic |

**Tooling by profile:**
- `saas-startup`: self-hosted (`unleash`, or a `feature_flags` Postgres table + simple percentage-rollout hashing function) is often enough — don't add a $500+/mo SaaS flag platform bill to a $8K/mo cloud ceiling before you need targeting rules beyond "percentage of users."
- `enterprise-scale`: LaunchDarkly/Split.io earn their cost once you need audited flag-change history, org-wide flag governance, and targeting rules across multiple services — the audit trail itself is often the compliance-driven reason to pay for a managed platform.

**The named discipline that prevents flag rot:** every release flag gets a tracked removal ticket the moment it's created, not the moment someone remembers. A codebase with dozens of permanently-true `if (flags.oldFeature)` checks two years later is flag-debt indistinguishable from dead code — worse, because it's dead code that still *looks* conditional. The Code Quality Analyzer's documentation-quality and complexity scoring will flag deeply nested flag checks as complexity debt; that's a real signal to schedule flag cleanup, not noise to ignore.

---

## 3. Monorepo tooling: Turborepo vs. Nx

Relevant the moment a `modular monolith` (see `architecture_patterns.md` §1) grows into multiple deployable apps sharing packages — e.g., a web app + a marketing site + a shared UI library + a shared types package.

| | Turborepo | Nx |
|---|---|---|
| **Model** | Thin, fast task-orchestration + remote caching layer on top of your existing package manager | Full monorepo framework — generators, plugins, dependency-graph visualization, enforced module boundaries |
| **Config weight** | Minimal — a `turbo.json` with task pipeline + cache config | Heavier — `nx.json`, project-level `project.json`, plugin ecosystem to learn |
| **Boundary enforcement** | Not built in — pair with `eslint-plugin-boundaries` yourself | Built in — `enforce-module-boundaries` ESLint rule ships with tags-based boundary rules |
| **Remote caching** | Vercel-hosted (free tier) or self-hosted | Nx Cloud (own hosted caching + distributed task execution) |
| **Pick when** | Small-to-mid monorepo (a handful of apps/packages), team wants minimal config overhead, already on Vercel | Larger monorepo, want generators for scaffolding new libs consistently, want built-in dependency-graph enforcement across many packages |
| **Named tradeoff** | You'll hand-roll boundary rules and generators; simplicity is the entire value proposition | Steeper initial learning curve and more opinionated structure; that structure is what pays off once you have 15+ internal packages and need to stop people from creating circular deps |

**Default for `saas-startup` profile:** Turborepo. The profile's whole premise (< 10 eng, modular monolith) means the monorepo is small enough that Nx's generator/graph machinery is more ceremony than the team needs yet.

**Default for `enterprise-scale` profile:** Nx, specifically for its enforced module-boundary tagging — at 50+ engineers across domain-bounded services (`architecture_patterns.md` §1), you want the build tool itself to reject a PR that imports across a boundary it shouldn't, not rely on code review catching it every time.

**What either tool actually buys you, concretely:**

```
turbo.json (pipeline + caching):
{
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".next/**"] },
    "test":  { "dependsOn": ["build"] },
    "lint":  {}
  }
}
```

Running `turbo run build` only rebuilds packages whose inputs changed (content-hash-based), and pulls from remote cache if another CI run or teammate already built the identical inputs — this is what keeps CI fast as the monorepo grows, not just a convenience.

---

## 4. Local dev environment parity with prod

The single most common source of "works on my machine" incidents: local dev running a different Postgres version, missing environment variables, or a Node version mismatch from what's deployed.

**Docker Compose as the parity mechanism** (default for `saas-startup`/`enterprise-scale` profiles):

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports: ["3000:3000"]
    env_file: .env.local
    depends_on: [db, redis]
  db:
    image: postgres:16       # ← pin to the SAME major version as prod RDS/Neon
    environment:
      POSTGRES_PASSWORD: devpass
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
volumes:
  pgdata:
```

**Parity checklist that actually prevents incidents:**
1. **Same major DB version locally as in prod** — a Postgres 14→16 feature gap (or a MySQL vs. Postgres mismatch if prod migrated and local didn't) is a recurring, entirely avoidable class of "works locally, breaks in prod" bug.
2. **`.env.example` checked into the repo, `.env.local` gitignored** — every required environment variable documented with a placeholder, so onboarding a new engineer means `cp .env.example .env.local` + fill in secrets, not spelunking through Slack history.
3. **Seed data script, not a prod DB dump** — `scripts/seed.ts` that creates realistic fixture data. Never let "pull down a prod snapshot" become the onboarding step, both for compliance (`data-sensitivity: pii-only`+ triggers real handling obligations) and because it silently couples local dev to prod data shape drifting out from under it.
4. **Preview environments per PR** — Vercel/Railway preview deploys, or Neon/PlanetScale branch-per-PR databases, so "does this change actually work against a real deployed environment" is answered before merge, not after. This is the parity mechanism that also derisks trunk-based development (§1): a flag-hidden, incomplete feature still gets tested against a real preview deploy before it lands on `main`.

**Node/tool version pinning:** `.nvmrc` or `package.json`'s `engines` field, enforced in CI (`nvm use` as a CI step, not just a local convention) — the goal is that a version mismatch fails CI loudly instead of manifesting as an intermittent bug three weeks later that only reproduces on one engineer's laptop.

---

## 5. CI/CD pipeline shape

A GitHub Actions pipeline that matches trunk-based + feature-flag development:

```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request, push]
jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version-file: '.nvmrc' }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:integration   # runs against the postgres service above

  e2e:
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - run: npx playwright test       # against a preview deploy URL, not localhost

  deploy-preview:
    needs: [lint-and-typecheck, test]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Vercel/Railway auto-deploys preview on PR open"
```

**Gate placement by profile:** `saas-startup` gates on lint+typecheck+unit tests before merge, runs E2E against the preview deploy, and trusts feature flags to de-risk incomplete work rather than blocking merge on "is this feature fully done." `enterprise-scale` (per-PR cadence with gates, per this skill's profile table) adds required security scanning (dependency CVE check, SAST) and a named human approver gate before production deploy — the "named-approver chain" the Decision Engine returns per profile is exactly this gate.

---

## 6. Testing strategy: the pyramid, applied

| Layer | What it covers | Speed | Volume |
|---|---|---|---|
| **Unit** | Pure business logic (domain layer from `architecture_patterns.md` §4) — no DB, no network | Milliseconds | Most of your tests |
| **Integration** | Repository implementations against a real (test) Postgres, API route handlers against a real DB | Seconds | Fewer than unit, covers the seams |
| **E2E** | Full user flow through a real browser against a deployed preview environment | Minutes | Fewest — cover critical paths only (signup, checkout, core workflow) |

**Common mistake this skill's Code Quality Analyzer flags as a documentation/coverage issue:** heavy E2E coverage with thin unit coverage. E2E tests are slow, flaky under CI parallelism, and expensive to debug when they fail (which layer broke?). The pyramid shape — many unit, some integration, few E2E — exists because it's the cost-effective way to get confidence without a CI run that takes 40 minutes and fails intermittently on network flake.

---

## 7. Code review process

**PR template that enforces the skill's own discipline (verifiable success criteria):**

```markdown
## What changed
## Why
## Verifiable impact
- [ ] API latency impact measured (p50/p95/p99) or N/A
- [ ] Frontend perf impact measured (LCP/INP/CLS) or N/A
- [ ] Feature flag added? Flag name + removal ticket link, or N/A
## Testing
- [ ] Unit tests added/updated
- [ ] Ran against a preview deploy
```

**Review checklist, in priority order:** correctness → security (secrets, injection, auth boundary checks — the same categories the Code Quality Analyzer scores) → architecture-boundary compliance (does this PR violate the module boundaries from `architecture_patterns.md` §1?) → style/nits last, and ideally auto-enforced by lint rather than spent as human review time.

---

## 8. Deployment strategies: blue-green, canary, feature-flag rollout

| Strategy | How it works | Pick when | Rollback speed |
|---|---|---|---|
| **Blue-green** | Two full production environments; switch traffic (via load balancer/DNS) from old (blue) to new (green) all at once | Need instant, full rollback capability; infra cost of running two full environments is acceptable | Instant — flip the switch back |
| **Canary** | New version gets a small % of traffic (1% → 10% → 50% → 100%), monitored at each step | Want to catch regressions on real traffic before full exposure, without doubling infra cost | Fast — pull the canary, but the % already exposed saw the bug |
| **Feature-flag rollout** | Code is deployed to 100% of instances already; the *feature* is exposed to an increasing % of users via flag targeting, independent of deploy | Want to decouple "is the code deployed" from "is the feature visible" — the norm for trunk-based dev (§1, §2) | Instant — flip the flag off, no redeploy needed |

**How these compose in practice for the `saas-startup`/`enterprise-scale` default:** deploy strategy (canary, via your platform's traffic-splitting) handles "is this build stable at the infrastructure level" (crashes, error rates, latency regressions); feature-flag rollout handles "is this feature ready for users" independently. A canary deploy can catch a build that 500s on startup; a feature flag catches "the checkout flow works technically but converts worse than the old one" — two different failure modes, two different tools, and conflating them is a common workflow mistake.

---

## 9. Monitoring and observability baseline

Every deploy strategy and every verifiable success criterion this skill requires (API p50/p95/p99, LCP/INP/CLS, uptime/SLO) is unmeasurable without this layer wired up before the first production incident, not during it.

| Signal | Tool examples | Tied to |
|---|---|---|
| **Structured logs** | Pino/Winston (Node), structlog (Python) → shipped to Datadog/Axiom/CloudWatch | Debugging, audit trail |
| **Metrics** | Prometheus + Grafana, or Datadog APM | The p50/p95/p99 latency numbers this skill requires on every recommendation |
| **Frontend RUM** | Vercel Analytics, Sentry Performance, or web-vitals library shipped to your own metrics pipeline | LCP/INP/CLS — the frontend perf target this skill requires |
| **Health checks** | `/healthz` endpoint checked by the load balancer/orchestrator | Uptime/SLO measurement, and what a canary rollout (§8) watches during a rollout step |
| **Error tracking** | Sentry, Bugsnag | Feeds directly into the P0/P1/P2 triage this skill's Workflow 2 (Audit Existing Codebase) runs |

**Minimum baseline for the `internal-tool` profile** (budget-constrained, low DAU): structured logs + a `/healthz` endpoint is often genuinely sufficient — full APM/RUM tooling is over-instrumentation for a tool with < 100 daily users and a $500/mo ceiling. **Minimum baseline for `enterprise-scale`:** all five rows, wired to a named on-call rotation with escalation policy — this is the operational half of the "named-approver chain" the Decision Engine returns for that profile.

# Tech Stack Guide

Concrete stack-selection guidance for the four customization profiles this skill ships (`saas-startup`, `enterprise-scale`, `internal-tool`, `marketing-site`). This is not a "pick your favorite" survey — it's a decision matrix with kill criteria, tied to the four forcing assumptions (team size, cadence, user-facing, budget) and the three verifiable success criteria (API latency, frontend perf, uptime/SLO) that every recommendation from this skill must carry.

Use this guide from Workflow 3 (Stack Selection) after the Decision Engine has returned a profile — this document explains *why* the profile maps to a given stack, so you can defend or override the recommendation with the team.

---

## 1. Frontend meta-framework decision matrix

| Framework | Rendering model | Pick when | Avoid when | Named tradeoff |
|---|---|---|---|---|
| **Next.js (App Router)** | RSC + hybrid SSR/SSG/ISR | SEO matters, team ships daily+, needs a large hiring pool, wants Vercel-managed edge | Team is < 3 eng and doesn't need SSR; app is pure SPA behind auth | Biggest ecosystem and hiring pool, but RSC's server/client boundary is a real learning curve — expect 2-3 sprints of "why is this a client component" churn. Cold-start on serverless functions (~200-400ms) hurts p99 if you're not on Vercel's edge runtime. |
| **Remix (React Router v7 framework mode)** | Server-rendered, nested loaders/actions | Data-heavy forms, want web-standards (FormData, fetch) over framework magic, deploying to non-Vercel infra (Fly.io, Cloudflare) | Team wants static-first marketing pages as the primary surface | Loader/action model maps cleanly to REST semantics — less "which cache layer did this go through" debugging than Next. Smaller plugin ecosystem than Next; fewer Stack Overflow answers when things break at 2am. |
| **SvelteKit** | Compiled, minimal runtime | Perf budget is aggressive (sub-100KB JS, low-end mobile), small team wants less boilerplate per feature | Team already has deep React hiring pipeline / component library investment (MUI, shadcn) | Ships less JS per route than any React framework — real win for LCP/INP on 4G. Cost: smaller talent pool, fewer third-party component libraries, and some enterprise procurement teams balk at "unfamiliar" stacks. |
| **Astro** | Islands architecture, zero-JS by default | Marketing site, content-heavy, mostly static, occasional interactive widget | Any app with meaningful client-side state or auth-walled dashboards | This is the `marketing-site` profile's default. Islands mean you opt into JS per-component instead of shipping a full app shell — best LCP numbers of any option here, but fighting the framework the moment you need shared client state across islands. |
| **React + Vite (SPA, no meta-framework)** | Client-only render | Internal tool, auth-walled dashboard, no SEO requirement, team wants fastest local dev loop | Anything public-facing and unauthenticated | This is the `internal-tool` profile's default when Retool/ 리테일-style no-code isn't sufficient. Vite HMR is the fastest dev loop of any option on this list. No SSR means no SEO, and you own the auth-gate-before-first-paint problem yourself. |

**Kill criterion:** if the answer to "SEO required?" is yes and the proposed stack is React+Vite SPA or Astro-without-SSR, stop — that combination cannot meet an LCP/CLS budget for crawled, unauthenticated pages. Route back to Next.js or Remix.

---

## 2. Backend framework decision matrix

| Framework | Language | Pick when | Avoid when | Named tradeoff |
|---|---|---|---|---|
| **Next.js API routes / Server Actions** | TS | Rapid prototype, `saas-startup` profile pre-Series-A, team is full-stack TS, no independent backend scaling need | Backend needs to scale/deploy independently of frontend, or business logic is heavy enough to warrant its own service boundary | Zero extra infra to stand up — one deploy, one repo. The tradeoff bites at scale: you can't scale API compute independently of page-render compute, and cold starts stack on top of frontend cold starts. |
| **FastAPI** | Python | API-first backend, team has Python/data-science overlap (ML inference, pandas pipelines), needs OpenAPI schema for free | Team is TS-only and doesn't want a second language in the stack | Async-native, Pydantic validation gives you request/response contracts for free (feeds directly into `api-design-reviewer` contract review). Best choice when the backend needs to share code with a data/ML team. |
| **Fastify** | Node/TS | Need raw throughput in Node, want a leaner alternative to NestJS without giving up a schema/plugin system | Team wants batteries-included DI/module system out of the box | Consistently benchmarks 2-3x Express throughput on JSON workloads. Smaller opinionated structure than NestJS means more upfront architectural decisions land on your team instead of the framework. |
| **NestJS** | TS | `enterprise-scale` profile — 50+ eng, regulated, needs enforced module boundaries and DI for testability | Small team, prototype speed matters more than long-term structure | Angular-style DI and decorators give you enforced module/service/controller separation, which is what lets 50+ engineers touch the same codebase without stepping on each other. Cost: verbose, more boilerplate per endpoint than Fastify or FastAPI, real ramp-up time for engineers coming from lighter frameworks. |
| **Django + DRF** | Python | Need auth, admin panel, ORM, and background jobs batteries-included on day one; content-heavy or internal-tool-adjacent apps | Team wants async-first request handling as the default (Django's async story is improving but still secondary) | Fastest path to a working CRUD admin + auth system of anything on this list — the Django admin alone can replace a `internal-tool` profile's entire custom UI in a pinch. Monolithic by design; splitting it later costs real migration effort. |

**Kill criterion:** if `data-sensitivity` is `pii-only` or higher and the proposed backend has no first-class request validation (raw Express with no schema layer), stop — add Zod/Pydantic-equivalent validation before scaffolding.

---

## 3. API contract style: REST vs GraphQL vs tRPC

| Style | Pick when | Avoid when | Tradeoff |
|---|---|---|---|
| **REST** | Public API, third-party consumers, need HTTP caching semantics (CDN, `Cache-Control`), `enterprise-scale` profile with external partners | Client needs deeply nested, variable-shape data from one round trip | Universally understood, cacheable at the HTTP layer, easy to version (`/v1`, `/v2`). Over-fetching/under-fetching on complex UIs pushes you toward BFF aggregation endpoints (see `architecture_patterns.md`). |
| **GraphQL** | Multiple client types (web + mobile + partner) with divergent data needs, want a typed schema as the contract of record | Single frontend, single backend, team < 5 eng — the resolver/schema tax isn't worth it yet | One endpoint, client-specified shape, strong tooling (codegen → typed hooks). Real costs: N+1 query risk unless you add DataLoader, cache invalidation is harder than REST's `Cache-Control`, and query complexity attacks need explicit depth/cost limiting. |
| **tRPC** | Full-stack TypeScript monorepo, one frontend consuming one backend, want end-to-end type inference with zero codegen step | Any non-TS client (mobile native, third-party API consumers) | Best DX of the three when both ends are TS in the same repo — autocomplete on the client for backend procedures, no schema file to keep in sync. Falls over the moment you need a non-TS consumer; it's explicitly not a public API technology. |

**Default for `saas-startup` profile:** tRPC if Next.js monolith with no external API consumers; REST the moment a mobile app or partner integration enters the picture.

**Default for `enterprise-scale` profile:** REST or GraphQL at service boundaries — tRPC's tight coupling to a single TS codebase doesn't fit a domain-bounded services architecture with polyglot services.

---

## 4. Database decision matrix

| Database | Pick when | Avoid when | Tradeoff |
|---|---|---|---|
| **PostgreSQL (self-managed or RDS/Cloud SQL)** | Default choice for anything with relational integrity needs, complex joins, transactions | Need horizontal write scaling beyond what read replicas + partitioning solve | The right default for 90% of `saas-startup` and `enterprise-scale` profiles. `JSONB` columns cover most "but we need flexible schema" arguments for Mongo without giving up transactions. |
| **PlanetScale / Neon / Supabase (managed Postgres/MySQL variants)** | `saas-startup` profile, team doesn't want to own backups/failover/patching, wants branch-per-PR database workflows | `enterprise-scale` with strict data-residency or air-gapped compliance requirements that managed platforms can't satisfy | PlanetScale's schema branching (deploy requests, non-blocking schema changes) is a genuine productivity unlock for fast-moving teams. Neon's branch-per-PR Postgres pairs naturally with preview-environment CI (see `development_workflows.md`). Cost is per-seat/per-branch pricing that gets real at scale, plus vendor lock-in on branching workflows. |
| **MongoDB** | Document-heavy, schema genuinely varies per record (event logs, CMS content, catalogs with heterogeneous attributes) | Financial/transactional data needing multi-document ACID guarantees as the primary workload | Schema flexibility is real for catalog/CMS-shaped data. Overused as a default when the actual access pattern is relational — the "we'll figure out the schema later" pitch causes query-pattern pain 12 months in. |
| **Redis** | Session store, rate limiting, cache-aside layer, pub/sub for real-time features, job queue backing (BullMQ) | Primary system of record for anything you can't afford to lose on a restart (without persistence configured) | Sub-millisecond reads make it the default cache-aside store (see `architecture_patterns.md` §5). Not a database of record unless you've explicitly configured AOF/RDB persistence and understand the durability tradeoff. |

**Kill criterion for `internal-tool` profile:** if DAU < 100 and someone proposes a bespoke Postgres + ORM + migrations setup, stop — that's the profile's named anti-pattern. Default to Retool/Airtable-backed data first; only stand up custom Postgres if the tool needs data shapes no no-code platform can express.

---

## 5. ORM / data-access layer

| ORM | Language | Pick when | Tradeoff |
|---|---|---|---|
| **Prisma** | TS | Want the most mature migration tooling + generated types in the Node ecosystem | Generated client is a black box for very complex queries; raw SQL escape hatch (`$queryRaw`) needed more often than the marketing suggests. Cold-start binary engine adds latency on serverless. |
| **Drizzle** | TS | Want SQL-like query builder with full type inference, no separate query engine binary, need to run at the edge (Cloudflare Workers) | Younger migration tooling than Prisma; smaller ecosystem of examples for edge-case schema patterns. |
| **SQLAlchemy (2.0 async)** | Python | FastAPI/Django-adjacent Python backend needing full ORM power, complex query composition | Steepest learning curve of any tool listed here; the 1.x → 2.0 async migration broke enough tutorials online that outdated examples are a real hazard. |

**Rule of thumb:** Drizzle for anything deploying to edge runtimes; Prisma for anything else in a TS backend; SQLAlchemy when the backend is already Python.

---

## 6. Authentication

| Solution | Pick when | Avoid when | Tradeoff |
|---|---|---|---|
| **Auth.js (NextAuth v5)** | Next.js app, standard OAuth providers + credentials, `saas-startup` profile budget-conscious | Need SSO/SAML for enterprise customers, or MFA/step-up auth out of the box | Free, self-hosted, good OAuth provider coverage. You own session/token edge cases and support burden as you scale past basic OAuth. |
| **Clerk** | `saas-startup` profile wants pre-built UI (sign-in, org switcher, MFA) shipped in days, budget allows $25-99+/mo tier | Cost-sensitive `internal-tool` profile ($500/mo ceiling) or data-residency requirements that conflict with Clerk's hosted user store | Fastest time-to-auth of anything on this list, including org/multi-tenancy primitives built in. Real per-MAU pricing at scale, and you're trusting a third party with your identity data plane. |
| **Custom JWT + refresh tokens** | `enterprise-scale` profile with existing IdP (Okta, Auth0, Azure AD) to federate against, or specific compliance requirements around token lifecycle | Small team without dedicated security review bandwidth — rolling your own auth is the single most common source of P0 findings in this skill's Code Quality Analyzer | Full control over token lifetime, rotation, and revocation semantics (see `architecture_patterns.md` §6 for the JWT + refresh token architecture). Highest implementation and audit burden of the three options — budget real security review time. |

**Kill criterion:** if `data-sensitivity` is `pii-only`+ and the plan is "roll our own JWT auth" without a named security reviewer, route to the `api-design-reviewer` / security review fork before scaffolding.

---

## 7. Deployment platform

| Platform | Pick when | Avoid when | Tradeoff |
|---|---|---|---|
| **Vercel** | Next.js/Remix frontend, `saas-startup` or `marketing-site` profile, want zero-ops preview deployments per PR | Backend needs long-running processes, WebSocket servers, or heavy background jobs beyond function timeouts | Best-in-class preview-deploy-per-PR workflow (pairs directly with the trunk-based + feature-flag workflow in `development_workflows.md`). Function timeout limits (10-300s depending on plan) and per-invocation pricing bite for compute-heavy workloads. |
| **Railway / Render / Fly.io** | Need a persistent backend process (WebSocket server, worker queue consumer), `saas-startup`/`internal-tool` budget tier, want Heroku-like simplicity without Heroku pricing | `enterprise-scale` profile needing VPC peering, dedicated compliance certs, or multi-region active-active | Genuinely simple `git push` deploys with persistent compute, reasonable pricing at small scale. Less mature multi-region and compliance tooling than AWS/GCP at the `enterprise-scale` tier. |
| **AWS / GCP (ECS/Fargate, Cloud Run, EKS/GKE)** | `enterprise-scale` profile, regulated industry, need VPC control, existing enterprise cloud commitment/discount | `saas-startup` or `internal-tool` profile — the ops overhead (IAM, networking, observability wiring) is the named anti-pattern here: you'll spend eng time on infra instead of product | Full control over networking, compliance posture, and scaling — this is what a dedicated platform team is for. The tradeoff is exactly that: it requires a platform team. Standing this up with < 10 eng is the most common "resume-driven architecture" mistake this skill's decision engine is built to prevent. |

---

## 8. Stack recommendations by use case (tie-back to profiles)

| Use case | Profile | Stack | Verifiable targets to state |
|---|---|---|---|
| MVP / pre-seed SaaS | `saas-startup` | Next.js (App Router) + tRPC + Postgres (Neon/Supabase) + Auth.js + Vercel | API p99 < 300ms, LCP < 2.5s on mobile-4G, uptime 99.5% |
| Regulated enterprise platform | `enterprise-scale` | NestJS or FastAPI services behind a BFF + Postgres (RDS, Multi-AZ) + custom JWT federated to corporate IdP + AWS ECS/EKS | API p99 < 200ms per service, LCP < 2.0s, uptime 99.95% with named on-call |
| Internal ops tool, < 100 DAU | `internal-tool` | Retool/Airtable first; if forced to custom: React+Vite SPA + lightweight Express/FastAPI + SQLite/Postgres, no public auth surface | API p99 < 500ms (internal tolerance), no public LCP requirement, uptime 99% (business-hours only acceptable) |
| SEO-dependent marketing site | `marketing-site` | Astro (static-first) or Next.js static export + headless CMS, Cloudflare Pages/Vercel | LCP < 1.8s, CLS < 0.1, uptime 99.9% (CDN-backed, near-zero write path) |

Every row above must ship with the three verifiable numbers filled in for the *actual* project, not copied verbatim — the Decision Engine (`scripts/fullstack_decision_engine.py`) prints the SLO floor for the matched profile; use that as the floor, not the ceiling.

# CI/CD Pipeline Guide

Detailed patterns, best practices, and anti-patterns for the pipelines this skill's `scripts/pipeline_generator.py` scaffolds (GitHub Actions and CircleCI), plus guidance for teams standardizing on GitLab CI.

## 1. Pipeline Stage Design

A well-designed pipeline has a strict dependency graph, not a flat script. The stages this skill scaffolds by default (`--stages=build,test,deploy`) map to this canonical shape:

```
lint/format (fast, fail-fast)
  └── unit test (parallelized by shard)
        └── build (artifact/image)
              └── security scan (SAST, image scan, SBOM)
                    └── integration test (against built artifact)
                          └── deploy staging (auto)
                                └── deploy production (gated: manual approval or auto-promote)
```

**Design rules:**

- **Fail fast, fail cheap.** Lint and static analysis (`eslint`, `ruff`, `tflint`) run first and take seconds — never let a 20-minute integration suite run before a syntax error is caught.
- **Test the artifact you ship, not source.** Integration tests should run against the built Docker image or compiled binary, not `npm run dev`. This is why `build` precedes `integration test` in the graph above.
- **One pipeline, many environments.** Avoid separate YAML files per environment (`ci-staging.yml`, `ci-prod.yml`); parameterize with environment inputs or reusable workflows instead. Divergent pipeline files rot independently and hide config drift.
- **Gate production explicitly.** Auto-deploy to staging; require either a manual approval, a passing canary window (see `deployment_strategies.md`), or a scheduled promotion window for production.

### GitHub Actions: job dependency graph

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint

  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm test -- --shard=${{ matrix.shard }}/4 --coverage

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        uses: docker/build-push-action@v5
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  deploy-staging:
    needs: security-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: |
          aws ecs update-service --cluster staging --service app-service --force-new-deployment

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production   # requires manual approval via GitHub Environment protection rules
    steps:
      - run: |
          aws ecs update-service --cluster production --service app-service --force-new-deployment
```

The `environment: production` key, combined with a required-reviewers protection rule configured in repo settings, is GitHub Actions' native manual-approval gate — no third-party action needed.

## 2. Caching Strategies

Caching is the highest-leverage lever for pipeline speed. Rules of thumb:

| Cache type | Key strategy | Tool support |
|---|---|---|
| Package manager (npm/pip/maven) | Hash of lockfile (`package-lock.json`, `poetry.lock`) | `actions/setup-node` `cache: npm`, `actions/cache` |
| Docker layers | BuildKit inline cache or registry cache | `docker/build-push-action` `cache-from`/`cache-to` |
| Compiled artifacts (Go, Rust) | Hash of source + compiler version | `actions/cache` on `~/.cargo`, `go build cache dir` |
| Terraform providers/modules | Hash of `.terraform.lock.hcl` | `actions/cache` on `.terraform` |

**GitHub Actions dependency cache (automatic, preferred over manual `actions/cache` when available):**
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # keys on package-lock.json hash automatically
```

**Docker BuildKit registry cache** (works across ephemeral runners, unlike local layer cache):
```yaml
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/org/app:${{ github.sha }}
    cache-from: type=registry,ref=ghcr.io/org/app:buildcache
    cache-to: type=registry,ref=ghcr.io/org/app:buildcache,mode=max
```

**CircleCI cache keyed on lockfile hash:**
```yaml
- restore_cache:
    keys:
      - v1-deps-{{ checksum "package-lock.json" }}
- run: npm ci
- save_cache:
    key: v1-deps-{{ checksum "package-lock.json" }}
    paths: [node_modules]
```

**Anti-pattern:** caching `node_modules` keyed only on branch name. Stale caches silently mask dependency changes and cause "works in CI, fails locally" bugs. Always key on a content hash of the lockfile, and bump the cache version prefix (`v1-` → `v2-`) when the cache format changes.

## 3. Matrix Builds

Use matrices to parallelize orthogonal variables — OS, language version, or test shard — without duplicating job definitions.

```yaml
jobs:
  test:
    strategy:
      fail-fast: false        # let all matrix legs finish so you see every failure, not just the first
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: ['18', '20']
        exclude:
          - os: macos-latest
            node: '18'         # skip combos that don't need coverage
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with: { node-version: ${{ matrix.node }} }
      - run: npm ci && npm test
```

**Sharding for large test suites** (combine matrix + a test splitter like `--shard` or `knapsack`):
```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4, 5, 6, 7, 8]
steps:
  - run: npm test -- --shard=${{ matrix.shard }}/8
```

Decision criteria: reach for a matrix when total test time exceeds ~5 minutes serially, or when you must certify against >1 runtime version/OS combo before release. Don't matrix trivially fast jobs (lint) — the runner-spin-up overhead (~10-20s) exceeds the savings.

**CircleCI equivalent** uses parallelism + `circleci tests split`:
```yaml
jobs:
  test:
    parallelism: 4
    steps:
      - run: |
          TESTFILES=$(circleci tests glob "test/**/*.spec.js" | circleci tests split --split-by=timings)
          npx jest $TESTFILES
```

## 4. Secrets Handling

**Never** put secrets in plaintext YAML, commit history, or Docker build args (they persist in image layer history). Rules by platform:

- **GitHub Actions:** use repository/environment `secrets`, referenced as `${{ secrets.AWS_ACCESS_KEY_ID }}`. Prefer OIDC federation over long-lived keys:
```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
      aws-region: us-east-1
```
  This eliminates static AWS keys entirely — GitHub issues a short-lived OIDC token the role trusts.

- **CircleCI:** use Contexts for shared secrets scoped to a security group, not project-level env vars for anything sensitive:
```yaml
workflows:
  deploy:
    jobs:
      - deploy:
          context: production-secrets
```

- **GitLab CI:** use masked + protected CI/CD variables, and protected branches so secrets never populate on unprotected refs/MRs from forks.

- **Cross-platform baseline:**
  - Rotate secrets on a schedule and immediately on suspected exposure.
  - Never `echo` a secret in a step (it prints to logs even if the var is masked, if concatenated into another string).
  - Use a secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) as the source of truth; CI-native secret stores should hold only the credential needed to *fetch* from the manager, not the application secrets themselves, for anything beyond small teams.
  - Scan for committed secrets in the pipeline itself: `gitleaks detect --source . --exit-code 1` as an early stage step.

## 5. Security Scanning Stage

Insert a scanning stage after build, before deploy:

```yaml
security-scan:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - name: SBOM + vulnerability scan
      uses: aquasecurity/trivy-action@0.24.0
      with:
        image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        severity: CRITICAL,HIGH
        exit-code: '1'
    - uses: github/codeql-action/upload-sarif@v3
      with: { sarif_file: trivy-results.sarif }
```

Fail the build on CRITICAL/HIGH, but track MEDIUM/LOW as tickets rather than blocking — otherwise teams disable scanning entirely out of frustration within a quarter.

## 6. Anti-Patterns

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| One giant job with 40 sequential `run` steps | No parallelism, unclear failure attribution, slow re-runs | Split into dependency-graphed jobs (see §1) |
| `latest` tag deployed to production | Non-reproducible, can't correlate incident to a commit | Tag images with git SHA or semver; `latest` for local dev only |
| Manually SSHing to "fix" a failed deploy | Breaks the audit trail, causes drift `terraform plan` will fight later | Fix the pipeline/IaC and redeploy through it |
| Secrets baked into Docker image via `ARG` | Visible in `docker history` even after later layers | Inject at runtime via env vars/secret mounts, never as build args |
| No caching, full `npm ci`/`terraform init` every run | 3-10x slower pipelines, higher CI spend | Apply §2 caching keyed on lockfiles |
| Skipping tests on hotfix branches "just this once" | Normalizes bypassing the gate; hotfixes are exactly when regressions are costliest | Keep the same gate for hotfix branches; shrink scope, not rigor |
| Deploy step with no health-check gate | Bad revision reaches 100% traffic before anyone notices | Use `scripts/deployment_manager.py`'s health-check gate or equivalent readiness probe before shifting traffic |

## 7. Pipeline-to-Deployment Handoff

The pipeline's `deploy` job should call the deployment strategy tooling, not re-implement rollout logic inline:

```yaml
deploy:
  needs: security-scan
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  environment: production
  steps:
    - run: |
        python scripts/deployment_manager.py deploy \
          --env=production \
          --image=app:${{ github.sha }} \
          --strategy=blue-green \
          --health-check-url=https://app.example.com/healthz
```

See `deployment_strategies.md` for how the health-check gate and traffic-shift math behind this call work, and `infrastructure_as_code.md` for how the underlying ECS/Kubernetes infrastructure the pipeline deploys to is provisioned and kept in sync.

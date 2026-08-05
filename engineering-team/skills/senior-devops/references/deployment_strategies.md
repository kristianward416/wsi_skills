# Deployment Strategies Guide

Deployment configs, security considerations, and scalability guidance for the strategies `scripts/deployment_manager.py` supports (`--strategy=blue-green|rolling`), plus canary and feature-flag patterns for teams extending beyond this skill's built-in strategies.

## 1. Blue-Green Deployment

Two full environments ("slots") run simultaneously — one live (`blue`), one idle staging the new version (`green`). Traffic switches atomically once the idle slot passes health checks.

**Manifests** (matches what `deployment_manager.py deploy --strategy=blue-green` generates):
```yaml
# k8s/deployment-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  labels: { app: myapp, slot: blue }
spec:
  replicas: 3
  selector:
    matchLabels: { app: myapp, slot: blue }
  template:
    metadata:
      labels: { app: myapp, slot: blue }
    spec:
      containers:
        - name: app
          image: ghcr.io/org/app:1.2.2   # current live version
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
```
```yaml
# k8s/deployment-green.yaml — identical except:
metadata:
  name: app-green
  labels: { app: myapp, slot: green }
spec:
  selector:
    matchLabels: { app: myapp, slot: green }
  template:
    metadata:
      labels: { app: myapp, slot: green }
    spec:
      containers:
        - name: app
          image: ghcr.io/org/app:1.2.3   # candidate version
```

**Rollout sequence:**
```bash
# 1. Deploy green with the new image, blue keeps serving all traffic
kubectl apply -f k8s/deployment-green.yaml
kubectl rollout status deployment/app-green -n production

# 2. Verify green health directly (bypassing the shared Service) before any traffic shift
kubectl port-forward deployment/app-green 8081:8080 -n production &
curl -sf http://localhost:8081/healthz || echo "GREEN UNHEALTHY — abort"

# 3. Atomic switch: repoint the Service selector from blue to green
kubectl patch service app-svc -n production \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# 4. Watch error rate/latency for a soak window (5-15 min) before decommissioning blue
```

**Rollback** is just re-patching the selector back to `blue` — sub-second, no rebuild, no redeploy:
```bash
kubectl patch service app-svc -n production -p '{"spec":{"selector":{"slot":"blue"}}}'
```

**Tradeoffs:** requires 2x steady-state compute during the overlap window; best for stateless services where an instant all-or-nothing cutover is acceptable and rollback speed matters more than gradual exposure.

## 2. Canary Deployment (Progressive Traffic Shift)

Canary routes a small, increasing percentage of live traffic to the new version while the rest stays on the stable version, gated by real production metrics at each step.

**Traffic-shift math.** A typical conservative canary schedule for a service doing 10,000 req/min:

| Stage | % to canary | Req/min to canary | Min sample size for signal* | Soak time |
|---|---|---|---|---|
| 1 | 1% | 100 | ~100 (borderline for <1% error-rate deltas) | 10 min |
| 2 | 5% | 500 | Solid for detecting >0.5% absolute error-rate change | 10 min |
| 3 | 25% | 2,500 | Solid for detecting >0.1% absolute error-rate change | 15 min |
| 4 | 50% | 5,000 | Full statistical confidence for most SLOs | 15 min |
| 5 | 100% | 10,000 | Full rollout, blue-green-style instant fallback still available | monitor |

*Rule of thumb for minimum sample size to detect an error-rate regression from baseline `p0` to `p1` at 95% confidence: `n ≈ 16 * p0*(1-p0) / (p1-p0)^2` (normal approximation to a two-proportion z-test). Example: baseline error rate 0.5% (`p0=0.005`), and you want to detect a jump to 1.5% (`p1=0.015`, delta 0.01): `n ≈ 16 * 0.005*0.995 / 0.0001 ≈ 796` requests — meaning stage 1 at 100 req/min needs roughly 8 minutes just to accumulate enough canary requests to statistically distinguish that regression from noise, which is why stage 1's soak time is set to 10 minutes, not 2.

**Kubernetes-native canary via two Deployments + weighted Service is crude** (Service load-balancing isn't precisely weighted); for real percentage control use a service mesh or ingress with traffic-splitting:

**Argo Rollouts canary (works with ArgoCD for GitOps-driven progressive delivery):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 1
        - pause: { duration: 10m }
        - setWeight: 5
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 15m }
        - setWeight: 50
        - pause: { duration: 15m }
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1          # run analysis starting at the first traffic shift
        args:
          - name: service-name
            value: app-svc
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: app
          image: ghcr.io/org/app:1.2.3
```

**Automated analysis gate** (Argo Rollouts `AnalysisTemplate` querying Prometheus — abort automatically if error rate regresses):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args: [{ name: service-name }]
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result[0] <= 0.01     # abort if error rate exceeds 1%
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status=~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

**When to use canary over blue-green:** when you need real production traffic (not synthetic health checks) to validate the release, when the blast radius of a bad release must be capped below 100% even briefly, or when the change is risky enough (major dependency bump, new algorithm) to want statistical confidence at each exposure level rather than a binary switch.

## 3. Rolling Update

The Kubernetes default — replaces old pods with new ones incrementally, no second full environment required.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 pod down at a time (16% of capacity for 6 replicas)
      maxSurge: 2          # up to 2 extra pods during rollout (33% temporary overcapacity)
  selector:
    matchLabels: { app: myapp }
  template:
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: app
          image: ghcr.io/org/app:1.2.3
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 15
            periodSeconds: 10
```

Trigger and monitor via `scripts/deployment_manager.py deploy --strategy=rolling` or directly:
```bash
kubectl set image deployment/app app=ghcr.io/org/app:1.2.3 -n production
kubectl rollout status deployment/app -n production
```

**Tradeoffs vs blue-green/canary:** no extra steady-state compute, but rollback is a re-rollout (slower than blue-green's instant selector flip), and both old and new versions serve traffic simultaneously during the rollout — only safe if the new version is backward-compatible at the API/schema level with the old one still running.

## 4. Feature Flags

Feature flags decouple *deployment* (code reaching production) from *release* (code becoming active for users), letting you deploy continuously and control exposure independently of the deployment strategy above.

**Pattern — flag-gated code path with a provider like LaunchDarkly, Unleash, or a simple config-service:**
```javascript
if (await flags.isEnabled('new-checkout-flow', { userId: user.id })) {
  return renderNewCheckout(user);
}
return renderLegacyCheckout(user);
```

**Why combine flags with canary/blue-green rather than replace them:** a deployment strategy controls which *code* is running on which fraction of infrastructure; a feature flag controls which *users* see a behavior regardless of which pods they hit. Use both — e.g., roll the new binary out via canary for infra-level safety (crashes, memory leaks), and gate the actual feature behind a flag for business-level control (targeted rollout to 5% of enterprise accounts, instant kill switch without a redeploy).

**Flag hygiene:** every flag needs an owner and a removal date at creation time. Flags that outlive their rollout become permanent branches in the code and a combinatorial testing burden — treat "flag cleanup" as part of the deploy ticket's definition of done, not a someday task.

## 5. Rollback Criteria and Automation

Define rollback triggers *before* deploying, not during an incident:

| Criterion | Threshold (typical) | Action |
|---|---|---|
| HTTP 5xx rate | > 1% absolute increase over 5-min pre-deploy baseline | Auto-abort canary / auto rollback |
| p99 latency | > 20% increase sustained for 3 min | Auto-abort canary |
| Health check endpoint | Fails 3 consecutive probes | Block traffic shift (blue-green) / pause rollout (rolling) |
| Crash-loop / OOMKill count | > 0 in candidate pods during soak | Immediate rollback, no soak wait |
| Business metric (e.g., checkout completion rate) | > 10% drop over 10-min window | Manual review, likely rollback |

**Automated rollback via this skill's tooling:**
```bash
python scripts/deployment_manager.py rollback --env=production --to-version=1.2.2
```

**kubectl-native rollback** (works for rolling and, with the blue/green Service selector reverted, blue-green too):
```bash
kubectl rollout undo deployment/app -n production
kubectl rollout status deployment/app -n production
kubectl get pods -n production -l app=myapp
curl -sf https://app.example.com/healthz || echo "ROLLBACK FAILED — escalate"
```

**Automating the gate, not just the rollback command** — wire the health check and abort into the pipeline step itself so a human doesn't have to notice the regression first:
```yaml
- name: Deploy with automated health gate
  run: |
    python scripts/deployment_manager.py deploy \
      --env=production --image=app:${{ github.sha }} \
      --strategy=blue-green --health-check-url=https://app.example.com/healthz
- name: Verify and auto-rollback on failure
  run: |
    for i in {1..10}; do
      if curl -sf https://app.example.com/healthz; then exit 0; fi
      sleep 15
    done
    echo "Health check failed after 150s — rolling back"
    python scripts/deployment_manager.py rollback --env=production --to-version=${{ github.event.before }}
    exit 1
```

## 6. Choosing a Strategy

| Situation | Recommended strategy |
|---|---|
| Stateless service, need instant rollback, can afford 2x compute briefly | Blue-green |
| High-traffic service, risky change, need statistical confidence before full exposure | Canary with automated analysis (Argo Rollouts) |
| Routine low-risk release, backward-compatible API, cost-sensitive | Rolling update |
| Business-level control over exposure independent of infra rollout | Feature flags layered on top of any of the above |
| Database schema change | Never any of the above alone — use expand/contract migrations (additive schema change, deploy code that writes both, backfill, deploy code that reads new only, drop old) regardless of deployment strategy |

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Blue-green switch succeeds but old sessions break | Stateful sessions pinned to old pods (sticky sessions, in-memory state) | Externalize session state (Redis) before adopting blue-green for this service |
| Canary analysis never passes even though app looks fine | Prometheus query window too short relative to traffic volume, causing false positives | Widen `interval`/lookback window, cross-check the traffic-shift math in §2 for minimum sample size |
| Rolling update stuck at "1 pod not ready" | New image failing readiness probe (bad config, missing secret) | `kubectl describe pod` for probe failure reason; roll back rather than waiting indefinitely |
| Rollback command succeeds but errors persist | Rollback reverted the deployment but not a paired schema migration | Schema changes need their own backward-compatible rollback plan (§6) — code rollback alone doesn't undo a forward-only migration |
| Feature flag "kill switch" doesn't stop the bleeding | Flag check cached client-side or flag evaluated only at process start, not per-request | Ensure flag SDK polls/streams updates and gating happens per-request, not at boot |

Cross-reference `cicd_pipeline_guide.md` for how the pipeline's `deploy` job invokes these strategies, and `infrastructure_as_code.md` for how the underlying compute (ECS service, Kubernetes cluster) these strategies target is provisioned.

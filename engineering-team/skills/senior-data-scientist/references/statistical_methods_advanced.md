# Advanced Statistical Methods

Reference companion to the Senior Data Scientist skill. Covers the failure modes and formal tools that separate rigorous statistical analysis from p-value theater: hypothesis testing pitfalls, Bayesian vs. frequentist tradeoffs, causal inference basics, and power analysis. Pairs with the A/B test workflow (`calculate_sample_size`, `analyze_experiment`) and the DiD workflow (`diff_in_diff`) in SKILL.md.

## 1. Hypothesis Testing Pitfalls

### 1.1 The Multiple Comparisons Problem

Every additional hypothesis test inflates the family-wise error rate (FWER) — the probability of at least one false positive across the family of tests.

```
FWER = 1 - (1 - alpha)^m
```

For `alpha = 0.05` and `m = 20` metrics tested (a typical "let's just check everything" dashboard review):

```
FWER = 1 - 0.95^20 ≈ 0.64
```

A 64% chance of at least one spurious "significant" result. This is the statistical mechanism behind most "the new button color increased revenue!" findings that fail to replicate.

**Corrections, in order of conservatism:**

- **Bonferroni**: `alpha_corrected = alpha / m`. Simple, controls FWER, but overly conservative when tests are correlated (common with related product metrics) — inflates Type II error.
- **Holm-Bonferroni** (step-down): sort p-values ascending `p_1 ≤ p_2 ≤ ... ≤ p_m`, compare `p_i` to `alpha / (m - i + 1)`. Uniformly more powerful than plain Bonferroni, same FWER guarantee.
- **Benjamini-Hochberg (FDR control)**: controls the *false discovery rate* (expected proportion of false positives among rejected hypotheses), not FWER — appropriate when you can tolerate a controlled fraction of false positives in exchange for power (e.g., screening 200 features for a first pass). Sort p-values, find the largest `i` such that `p_i ≤ (i / m) * alpha`, reject all hypotheses `1..i`.

```python
from statsmodels.stats.multitest import multipletests

reject, p_adj, _, _ = multipletests(p_values, alpha=0.05, method="fdr_bh")
```

Rule of thumb for the skill's A/B checklist: if you pre-register one primary metric, you don't need correction on it. Any secondary/guardrail metrics you still report as "significant" need Bonferroni or BH applied — this is why the checklist says `alpha / n_metrics`.

### 1.2 P-Hacking and Researcher Degrees of Freedom

P-hacking is not usually fraud — it's the cumulative effect of small, individually reasonable analytic choices made *after* seeing the data:

- Stopping data collection once `p < 0.05` is reached (see sequential peeking, covered in `experiment_design_frameworks.md`).
- Trying several outcome metrics and reporting the one that "worked."
- Trying several covariate/segment cuts (by device, by region, by cohort) until one segment is significant.
- Excluding "outliers" post hoc in a way that happens to help the result.
- Choosing between one-tailed/two-tailed tests after seeing the direction of the effect.

Simmons, Nelson & Simonsohn (2011) showed that with 4 flexible researcher degrees of freedom (two DVs, optional covariate, optional extra 10 samples), the false positive rate can exceed 60% while every individual choice looks defensible in isolation.

**Mitigations:**
- Pre-register the primary metric, sample size, and stopping rule *before* looking at data (the skill's checklist step 1-2).
- Use a held-out confirmation sample for any exploratory finding — exploratory analysis generates hypotheses, it does not confirm them.
- Report effect sizes and CIs for *all* metrics examined, not just the significant one, so reviewers can judge multiplicity.
- Distinguish "confirmatory" analysis (pre-registered, one shot) from "exploratory" analysis (unlimited looks, no claims of significance) explicitly in the writeup.

### 1.3 Misreading What a P-Value Means

A p-value is `P(data this extreme or more | H0 true)`. It is **not**:
- `P(H0 true | data)` — that requires a prior (Bayes' theorem) and is a different quantity entirely.
- A measure of effect size or practical importance — with enough sample, trivial effects become "significant."
- A pass/fail gate that makes an effect real at `p=0.049` and unreal at `p=0.051`.

Always pair a p-value with a confidence interval and an effect size (lift, Cohen's d, odds ratio). This is why `analyze_experiment()` in the skill returns `lift`, `ci_95`, *and* `p_value` — never ship just the p-value.

## 2. Bayesian vs. Frequentist Tradeoffs

| Dimension | Frequentist | Bayesian |
|---|---|---|
| Core object | Sampling distribution of a statistic under repeated experiments | Posterior distribution over the parameter given observed data and a prior |
| P-value / posterior meaning | `P(data \| H0)` | `P(parameter \| data)` — directly answers "how likely is treatment better" |
| Peeking / sequential looks | Invalidates error-rate guarantees unless corrected (alpha-spending, see below) | Naturally valid — posterior updates are always "honest," no fixed sample size required |
| Small samples | CIs can be unstable, normal approximations break down | Priors regularize estimates, but results are prior-sensitive |
| Interpretability for stakeholders | "We reject H0 at alpha=0.05" — often misread as "95% chance treatment wins" | "There's an 87% probability treatment beats control by ≥2%" — matches how people actually reason |
| Computation | Closed-form for standard tests (z-test, t-test) | Requires specifying a prior; closed-form for conjugate cases (Beta-Binomial for conversion rates), else MCMC/variational inference |
| Multiple comparisons | Requires explicit correction (Bonferroni/BH) | Hierarchical models partially pool estimates, naturally shrinking noisy comparisons — a soft alternative to correction |

### Practical Bayesian A/B example (Beta-Binomial conjugate)

For a binary conversion metric, put a `Beta(1,1)` (uniform) prior on each variant's true rate:

```python
import numpy as np
from scipy.stats import beta

def bayesian_ab_test(control_conv, control_n, treat_conv, treat_n, n_samples=200_000):
    # Posterior: Beta(alpha_prior + successes, beta_prior + failures)
    post_c = beta(1 + control_conv, 1 + control_n - control_conv)
    post_t = beta(1 + treat_conv, 1 + treat_n - treat_conv)
    samples_c = post_c.rvs(n_samples)
    samples_t = post_t.rvs(n_samples)
    prob_treatment_better = float(np.mean(samples_t > samples_c))
    expected_lift = float(np.mean((samples_t - samples_c) / samples_c))
    return {"P(treatment > control)": prob_treatment_better, "expected_relative_lift": expected_lift}
```

This can be monitored continuously without inflating error rates, which is why many product teams run Bayesian dashboards for exploration and reserve a frequentist fixed-horizon test (or a sequential design, see next doc) for the decision that ships.

**When to prefer which:**
- Frequentist: regulatory or high-stakes decisions where a pre-specified, auditable error-rate guarantee is required (clinical trials, financial reporting).
- Bayesian: continuous monitoring dashboards, small-sample or low-traffic experiments, decisions naturally framed as "probability X beats Y" for a non-technical stakeholder, or when informative priors from prior experiments meaningfully improve estimates (empirical Bayes / hierarchical shrinkage across many similar experiments, e.g. per-store or per-SKU tests).

## 3. Causal Inference Basics

Association is not causation because of **confounding** (a variable affects both treatment assignment and outcome) and **selection bias** (who ends up treated is not random). The three workhorse tools below each attack this from a different angle.

### 3.1 Propensity Score Matching / Weighting

Used when treatment was *not* randomized (observational data) but you can measure the confounders that plausibly drove who got treated.

1. Fit a model (logistic regression, gradient boosting) predicting treatment assignment `T` from covariates `X`: `e(X) = P(T=1 | X)` — the propensity score.
2. Match each treated unit to control unit(s) with similar `e(X)` (nearest-neighbor, caliper matching, e.g. within 0.02), **or** weight units by inverse propensity (IPTW): `w_i = T_i/e(X_i) + (1-T_i)/(1-e(X_i))`.
3. Check covariate balance post-matching: standardized mean difference should drop below 0.1 for every covariate.
4. Estimate the treatment effect on the matched/weighted sample (difference in means, or a weighted regression).

```python
from sklearn.linear_model import LogisticRegression
import numpy as np

def propensity_scores(X, treatment):
    model = LogisticRegression(max_iter=1000).fit(X, treatment)
    return model.predict_proba(X)[:, 1]

def iptw_ate(y, treatment, X, clip=(0.01, 0.99)):
    e = np.clip(propensity_scores(X, treatment), *clip)  # clip to avoid extreme weights
    w = treatment / e + (1 - treatment) / (1 - e)
    ate = np.sum(w * treatment * y) / np.sum(w * treatment) - np.sum(w * (1 - treatment) * y) / np.sum(w * (1 - treatment))
    return ate
```

**Caveat:** propensity methods only adjust for *observed* confounders. Unobserved confounding (e.g. motivation, unmeasured intent) is not fixed — this is the fundamental limitation versus randomization.

### 3.2 Instrumental Variables (IV)

Used when there's likely **unobserved** confounding between treatment and outcome, but you have an *instrument* `Z` that (a) affects treatment `T`, (b) affects the outcome `Y` *only through* `T` (exclusion restriction), and (c) is otherwise as-good-as-randomly assigned.

Two-Stage Least Squares (2SLS):

```
Stage 1: T_hat = alpha_0 + alpha_1 * Z + alpha_2 * X   (regress treatment on instrument + covariates)
Stage 2: Y     = beta_0  + beta_1  * T_hat + beta_2 * X (regress outcome on predicted treatment)
```

`beta_1` is the causal estimate (Local Average Treatment Effect, LATE — the effect for units whose treatment status the instrument actually moves, i.e. "compliers").

```python
from linearmodels.iv import IV2SLS
import pandas as pd

model = IV2SLS.from_formula("y ~ 1 + [treatment ~ instrument] + control1 + control2", data=df)
result = model.fit(cov_type="robust")
```

Classic examples: distance to nearest clinic as an instrument for treatment uptake; randomized encouragement/nudge as an instrument for actual usage of a feature. The exclusion restriction is untestable — it must be argued qualitatively, and weak instruments (`F-stat < 10` on the first stage) produce badly biased 2SLS estimates, so always report the first-stage F-statistic.

### 3.3 Difference-in-Differences (DiD)

Already in the skill's core workflow (`diff_in_diff`). The key identifying assumption is **parallel trends**: absent treatment, treated and control groups would have moved in parallel over time. This doc adds the diagnostics and extensions the core workflow references but doesn't spell out:

- **Pre-trend test**: regress outcome on group × each pre-period time dummy; none of the interaction coefficients should be significant. Plot the two groups' raw trends pre-period as a sanity check — a formal test can still miss a visually obvious divergence.
- **Clustering**: standard errors must be clustered at the unit of treatment assignment (e.g. store, market, user cohort), not the observation level, or you will dramatically understate SEs (this is why the skill's checklist says "cluster SEs at the unit level").
- **Staggered adoption**: when units are treated at different times (common in rollouts), the classic two-way fixed effects DiD estimator is biased in the presence of heterogeneous treatment effects (Goodman-Bacon 2021). Use the Callaway-Sant'Anna estimator or Sun-Abraham interaction-weighted estimator instead:

```python
# csdid-style estimator (conceptual — via R's `did` package or Python `csdid`)
# Estimates group-time ATT(g,t) then aggregates, avoiding "forbidden comparisons"
# between already-treated units used as controls.
```

- **Placebo/robustness**: run DiD on a fake pre-period cutoff (should show null effect) and on an outcome that shouldn't be affected by treatment (should also show null).

### 3.4 Choosing a Method

| Setting | Method |
|---|---|
| Randomized assignment exists | Simple difference in means / two-proportion z-test |
| Observational, confounders measured | Propensity score matching/weighting |
| Observational, suspected unobserved confounding, valid instrument available | Instrumental variables |
| Policy/treatment rolled out to some units at a point in time, panel data available | Difference-in-differences (with staggered-adoption correction if applicable) |
| No control group, single unit affected (e.g. one market's price change) | Synthetic control — construct a weighted combination of untreated units matching pre-period trend |

## 4. Power Analysis

Power (`1 - beta`) is the probability of detecting a true effect of a given size. Underpowered experiments are the single most common reason for "no significant difference" ≠ "no effect."

### 4.1 Two-Proportion Test (matches `calculate_sample_size` in SKILL.md)

```
n per arm = ((z_(1-alpha/2) + z_(1-beta))^2 * (p1(1-p1) + p2(1-p2))) / (p1 - p2)^2
```

- `z_(1-alpha/2) = 1.96` for `alpha=0.05` two-sided; `z_(1-beta) = 0.84` for `power=0.80`.
- Sensitivity: halving the MDE roughly **quadruples** the required sample size (inverse-square relationship) — this is the single most important intuition to give stakeholders who want to detect a 1% lift in two days.

### 4.2 Two-Sample T-Test (continuous metrics, e.g. revenue per user)

```
n per arm = 2 * ((z_(1-alpha/2) + z_(1-beta))^2 * sigma^2) / delta^2
```

where `sigma` is the (pooled) standard deviation and `delta` is the minimum detectable difference in means. Revenue/spend metrics are typically heavy-tailed, so estimate `sigma` from real historical data (or via bootstrap), not a normal-theory guess — underestimating variance is the most common power-analysis mistake in practice.

```python
import numpy as np
from scipy import stats

def sample_size_continuous(sigma, mde_absolute, alpha=0.05, power=0.8):
    z_a = stats.norm.ppf(1 - alpha / 2)
    z_b = stats.norm.ppf(power)
    n = 2 * ((z_a + z_b) ** 2) * (sigma ** 2) / (mde_absolute ** 2)
    return int(np.ceil(n))
```

### 4.3 Post-Hoc Power Is Not Useful

Computing "power" from the observed effect size *after* the experiment (post-hoc/observed power) is mathematically a deterministic function of the p-value and adds no information — a common reviewer red flag. If a test came back non-significant, don't compute post-hoc power to argue it was "underpowered but promising"; instead compute the CI on the effect and check whether it excludes a practically meaningful effect size.

### 4.4 Variance Reduction to Increase Effective Power Without More Sample

- **CUPED** (Controlled-experiment Using Pre-Experiment Data): adjust the outcome using a pre-period covariate `X` correlated with `Y`: `Y_cuped = Y - theta*(X - mean(X))`, with `theta = Cov(X,Y)/Var(X)`. Reduces variance by `1 - corr(X,Y)^2`, directly shrinking required sample size for the same MDE.
- **Stratified/blocked randomization** on known high-variance covariates (e.g. user tenure) reduces residual variance versus simple randomization.
- **Ratio/percentage metrics with high variance**: consider a paired or covariate-adjusted design instead of raw mean comparison.

## 5. Summary Checklist to Add to Any Analysis

1. State the primary metric and correction plan for secondary metrics *before* running the test.
2. Report effect size + 95% CI, never a bare p-value.
3. If continuously monitoring, use a sequential/Bayesian method, not repeated frequentist peeks.
4. For observational causal claims, name the identifying assumption explicitly (parallel trends, exclusion restriction, ignorability) and show at least one diagnostic for it.
5. Run power analysis with a *realistic* variance estimate before collecting data, not after.

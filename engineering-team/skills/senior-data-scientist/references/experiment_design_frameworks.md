# Experiment Design Frameworks

Reference companion to the Senior Data Scientist skill's A/B testing workflow (`calculate_sample_size`, `analyze_experiment`). Covers the mechanics of designing a trustworthy experiment: sample size and MDE tradeoffs, the sequential-testing/peeking problem, factorial designs, and switchback tests for settings with network/spillover effects.

## 1. A/B Test Design Fundamentals

### 1.1 The Sample Size / MDE / Power Triangle

Four quantities are linked — fixing three determines the fourth:

```
n per arm = ((z_(1-alpha/2) + z_(1-beta))^2 * (p1(1-p1) + p2(1-p2))) / (p1 - p2)^2
```

This is the formula behind the skill's `calculate_sample_size()`. Practical implications:

- **MDE and n are inversely quadratic**: to detect half the effect size, you need ~4x the sample. Always ask "what's the smallest lift that would change a business decision?" and size for *that*, not for "whatever we can detect in two weeks."
- **Baseline rate matters**: metrics near 50% require the largest samples for a given *relative* MDE (variance `p(1-p)` peaks at `p=0.5`); very low-baseline metrics (e.g. 0.5% conversion) need large absolute samples even for big relative lifts, because absolute event counts stay low.
- **Report relative vs. absolute MDE explicitly.** "5% MDE" is ambiguous — clarify whether it's `p2 = p1*1.05` (relative, what the skill's function uses) or `p2 = p1 + 0.05` (absolute, five percentage points — usually enormous and unrealistic for conversion-rate metrics).

### 1.2 Duration, Not Just Sample Size

Once `n` is known, `duration = n / (daily_traffic * allocation_fraction)`. Two additional constraints override the naive calculation:

- **Run at least one full business cycle** (the skill's checklist says ~2 weeks) to average over day-of-week effects — a 3-day test run Mon-Wed will overweight weekday-only behavior.
- **Novelty/primacy effects**: a new feature's effect often decays (novelty) or grows (primacy, as users learn it) over the first days. Don't call a test early even if `n` is reached in 3 days if the true rollout decision needs a stable long-run effect — check whether the daily lift trend has flattened, not just whether `n` was hit.

### 1.3 Sample Ratio Mismatch (SRM)

Before trusting any result, check:

```
SRM check: |n_control - n_treatment| / expected_split should be < ~1% in practice;
formally, run a chi-square goodness-of-fit test against the intended split (e.g. 50/50).
p < 0.001 on this test => investigate the randomization/logging pipeline before
trusting the experiment's substantive result.
```

SRM indicates a broken assignment or logging pipeline (e.g., a bot filter that only fires in one arm, a redirect that fails more often for one variant) and invalidates the result regardless of what the primary metric shows.

### 1.4 Randomization Unit

Randomize at the unit that matches how the treatment is experienced and how the outcome is measured — the skill's checklist flags "randomize at user, not session, level." Session-level randomization for a metric that accumulates across sessions (e.g. weekly retention) leaks a user across both arms and dilutes/contaminates the estimated effect. For marketplace/two-sided products, consider whether buyer-level or seller-level (or geography-level, see switchback tests below) randomization is the one that avoids interference.

## 2. Sequential Testing and the Peeking Problem

### 2.1 Why Peeking Inflates False Positives

A fixed-horizon test computes the p-value assuming a single look at the data at a pre-determined `n`. If you instead check the dashboard daily and stop the moment `p < 0.05` is crossed (optional stopping), the *effective* Type I error rate balloons — repeatedly testing a true null against a random walk of accumulating p-values means it will cross the 0.05 threshold by chance with much higher than 5% probability over many looks. With continuous monitoring under the null, the p-value process will cross any fixed threshold with probability approaching 1 given enough looks (law of the iterated logarithm) — the classic "peeking problem."

Simulated approximate inflation for a two-sided `alpha=0.05` test peeked at repeatedly:
- 5 looks: effective false-positive rate ≈ 0.14
- 10 looks: ≈ 0.19
- Continuous monitoring: → 1 as data accumulates indefinitely without correction

### 2.2 Fixes

**a) Pre-register a fixed horizon and don't look until then.** Simplest, but wastes information (can't stop early for a huge, obvious effect) and wastes traffic if you could have stopped sooner.

**b) Group sequential design with alpha-spending (O'Brien-Fleming / Pocock boundaries).** Decide the number of interim looks `K` in advance; allocate a shrinking p-value threshold to each look so the cumulative Type I error still sums to `alpha`.

- **Pocock**: roughly constant boundary across looks, e.g. for `K=5`, use `alpha_i ≈ 0.016` at each look (varies with exact spending function) — easier to stop early, costs power at the final look.
- **O'Brien-Fleming**: very strict early boundaries (near-impossible to stop at look 1), relaxing toward the nominal `alpha` at the final look — preferred in practice because it preserves power while still allowing early stopping for large effects.

```python
# Conceptual O'Brien-Fleming boundary at look k of K
# alpha_k spent such that cumulative spending equals alpha at k=K
import scipy.stats as stats

def obf_boundary_z(k, K, alpha=0.05):
    # Approximation: boundary shrinks toward z_(1-alpha/2) as k -> K
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    return z_alpha * np.sqrt(K / k)
```

**c) Always Valid Inference / sequential probability ratio test (mSPRT), used by tools like Optimizely Stats Engine.** Uses a test statistic (often a mixture-likelihood-ratio) that is valid to check *at any time*, at any sample size, without a pre-declared number of looks — the p-value/confidence sequence stays valid under continuous monitoring. This is the right tool when stakeholders insist on checking a live dashboard daily.

**d) Bayesian continuous monitoring** (see `statistical_methods_advanced.md` §2) — sidesteps the peeking problem entirely because posterior probabilities don't carry a repeated-testing error-rate guarantee to violate, though decision *rules* built on top of a posterior (e.g. "ship when P(better)>95%") still need their own calibration via simulation if used as a hard stopping rule.

### 2.3 Practical Recommendation

For the skill's core A/B workflow: state the fixed sample size and stop date in the experiment brief before launch (checklist step 2). If the business genuinely needs to react early, don't just eyeball the dashboard — switch to a named sequential method (group sequential or mSPRT) and pre-register *that* design, including the number/spacing of looks, so the error-rate guarantee is preserved.

## 3. Factorial Designs

Used to test multiple independent changes (e.g. headline copy × button color) in one experiment instead of running sequential single-factor tests, while also estimating **interaction effects** between factors.

### 3.1 2×2 Factorial

Four cells: `(A0,B0)`, `(A1,B0)`, `(A0,B1)`, `(A1,B1)`. Fit:

```
Y = beta_0 + beta_1*A + beta_2*B + beta_3*(A*B) + epsilon
```

- `beta_1`, `beta_2`: main effects of A and B.
- `beta_3`: interaction — is the effect of A different depending on B's state? A significant interaction means "the best combination isn't just picking each factor's individual winner" — e.g. a new headline might only lift conversion when paired with the new button, not with the old one.

```python
import statsmodels.formula.api as smf

model = smf.ols("conversion ~ A + B + A:B", data=df).fit(cov_type="HC3")
```

### 3.2 Sizing a Factorial Design

Each cell needs the same per-arm `n` as a standalone two-arm test would (computed from the main-effect MDE you care about) — a 2x2 design needs roughly **4x** the traffic of a single A/B test to detect the same main-effect MDE with the same power, because each factor's comparison only uses half the total sample (e.g. A1 cells vs A0 cells collapsing across B). Detecting the *interaction* term reliably typically requires a larger MDE assumption or even more sample than either main effect, since interaction effects are usually smaller than main effects in practice — size explicitly for the interaction if it's the decision-relevant effect, don't assume the main-effect sample size covers it.

### 3.3 When to Use vs. Avoid

- Use when: factors are cheap to combine (both are UI/config changes), you specifically care about interaction, or you want to test several independent levers without running sequential tests that burn calendar time.
- Avoid when: traffic is limited (cells become underpowered fast as factors multiply — a 2×2×2 design has 8 cells), or the factors are likely to have large, business-critical interactions that make "ship whichever main effect wins" strategically wrong to decide from an underpowered interaction estimate.

## 4. Switchback Tests (Network/Spillover Effects)

Standard user-randomized A/B tests break down when treatment on one unit spills over to affect other units in the *same* arm or the *opposite* arm — this is **interference**, and it violates the Stable Unit Treatment Value Assumption (SUTVA) that ordinary A/B analysis relies on.

### 4.1 When You Need One

- **Marketplace/two-sided effects**: a pricing or matching algorithm change for "treatment" riders/drivers changes the supply/demand balance seen by "control" riders/drivers in the same market (e.g. rideshare surge pricing, ad auction changes).
- **Shared inventory/capacity**: treatment and control compete for the same limited pool (delivery drivers, ad impressions, warehouse capacity) so improving one arm mechanically starves the other, biasing the comparison.
- **Geographic or temporal spillover**: word of mouth, shared infrastructure, or cross-market cannibalization.

### 4.2 Design

Instead of randomizing *users*, randomize **time × unit blocks** — typically a whole market (city, warehouse) is switched between treatment and control on a fixed cadence (e.g. every 2 hours, or every day), so at any instant the whole market is in one condition, avoiding contamination *within* a time block.

```
Market M1: [Control][Treatment][Control][Treatment]...  (alternating blocks)
Market M2: [Treatment][Control][Treatment][Control]...  (offset to average out time-of-day effects)
```

Key design choices:
- **Block length** must exceed the treatment's "washout" time (how long an effect persists after switching back) — too-short blocks bleed carryover effects between periods. Include a **buffer/burn-in period** at each switch that's excluded from analysis.
- **Randomize the *order*** of treatment/control blocks per market (not just alternate deterministically) to avoid confounding with time-of-day/day-of-week patterns — this is what makes it a valid randomized design rather than a simple pre/post comparison.
- Analyze with a model that includes market and time-block fixed effects, clustering standard errors at the market level (same clustering logic as DiD, since a switchback is structurally a within-market repeated-measures design):

```python
import statsmodels.formula.api as smf

model = smf.ols(
    "outcome ~ treatment + C(market) + C(time_block)",
    data=df
).fit(cov_type="cluster", cov_kwds={"groups": df["market"]})
```

### 4.3 Tradeoffs vs. User-Level A/B

| | User-randomized A/B | Switchback |
|---|---|---|
| Interference bias | High risk if network effects exist | Controlled — whole market shares one condition |
| Effective sample size | n = number of users | n = number of market-time blocks (usually much smaller — plan for lower power, longer run) |
| Analysis complexity | Two-proportion z-test | Panel regression with market + time fixed effects, clustered SEs |
| Best for | Independent-user products (most consumer apps) | Marketplaces, shared-capacity systems, pricing/matching algorithms |

## 5. Design Checklist (extends the skill's core A/B checklist)

1. Confirm the randomization unit avoids interference; if not, consider a switchback or cluster-randomized design instead of user-level.
2. Compute sample size from the *decision-relevant* MDE, and re-derive for factorial cells (×4 rule for 2×2) or switchback blocks (expect far fewer effective units than a user-level test).
3. Decide up front: fixed-horizon test, or a named sequential method with a pre-specified alpha-spending function — never "we'll just watch the dashboard and stop when it looks good."
4. For factorial designs, explicitly size for the interaction term if it's decision-relevant, not just the main effects.
5. For switchback tests, set block length longer than the expected carryover/washout period and randomize block order, not just alternate mechanically.

# Feature Engineering Patterns

Reference companion to the Senior Data Scientist skill's feature engineering workflow (`build_feature_pipeline`, `add_time_features`). Covers categorical encoding strategies and their leakage risks, time-series feature patterns (and how they leak from the future), and feature selection with concrete, checkable criteria.

## 1. Encoding Strategies for Categorical Features

The skill's `build_feature_pipeline` uses one-hot encoding for categoricals inside a `ColumnTransformer`. One-hot is safe and leakage-free but breaks down for high-cardinality columns (the skill's checklist already flags ">50 levels" as the trigger to switch away from it). Here's what to switch to, and the risk each option carries.

### 1.1 One-Hot Encoding
- Safe default: no target leakage possible, since it only uses the category label itself.
- Fails at high cardinality: a `zip_code` or `user_id`-like column with 5,000 levels produces 5,000 sparse columns, blows up model size, and most levels get too few samples to learn from.

### 1.2 Target Encoding (Mean Encoding) — and Its Leakage Risk

Replace each category with a statistic of the target computed over rows sharing that category (typically the mean):

```
encoded(category_c) = mean(y | category = c)
```

**The leakage risk**: if you compute this mean using *all* rows (including the row being encoded) and then train on it, you've leaked the label into the feature — for categories with few observations, the encoded value is nearly the target itself (extreme overfitting, especially visible for rare categories where `mean(y | c)` is computed from just 1-2 rows).

**Correct implementations:**

- **K-fold target encoding**: split training data into K folds; for each fold, compute the category means using only the *other* K-1 folds, and encode that fold with those out-of-fold means. Never touches a row's own label.
- **Leave-one-out target encoding**: for row `i`, `encoded_i = (sum(y | c) - y_i) / (count(c) - 1)`. Excludes the row's own label directly.
- **Smoothing / Bayesian shrinkage toward the global mean** — critical for low-count categories:

```
encoded(c) = (count(c) * mean(y|c) + k * global_mean) / (count(c) + k)
```

  `k` is a smoothing strength hyperparameter (e.g. `k=10-50`); as `count(c) -> 0`, the encoding shrinks to the global mean instead of overfitting to a handful of observations.

```python
from category_encoders import TargetEncoder
from sklearn.model_selection import KFold
import numpy as np

def kfold_target_encode(df, cat_col, target_col, n_splits=5, smoothing=20):
    encoded = np.zeros(len(df))
    kf = KFold(n_splits=n_splits, shuffle=True, random_state=42)
    global_mean = df[target_col].mean()
    for train_idx, val_idx in kf.split(df):
        means = df.iloc[train_idx].groupby(cat_col)[target_col].mean()
        counts = df.iloc[train_idx].groupby(cat_col)[target_col].count()
        smoothed = (counts * means + smoothing * global_mean) / (counts + smoothing)
        encoded[val_idx] = df.iloc[val_idx][cat_col].map(smoothed).fillna(global_mean).values
    return encoded
```

- **Always fit the encoding mapping (whichever variant) on train only, then apply to test/validation** — this is the same principle the skill's checklist states generically ("never fit transformers on the full dataset"); target encoding is simply the case where violating it does the *most* damage, because it directly encodes the label.

### 1.3 Embeddings for High-Cardinality Categoricals

For categoricals with complex, non-monotonic relationships to the target (e.g. `product_id`, `merchant_id`) and enough data to support it, learn a dense embedding jointly with the model (entity embeddings, as in neural nets, or via a gradient-boosted-tree-friendly proxy like frequency + target-encoded interactions):

- **Neural entity embeddings**: an embedding layer maps each category to a learned dense vector (typically `dim ≈ min(50, (cardinality+1)//2)` as a starting heuristic), trained end-to-end with the rest of the network. Captures similarity between categories (e.g. similar products cluster in embedding space) that one-hot cannot.
- **Pretrained embeddings** (e.g. from a co-occurrence model, word2vec-style on category sequences, or a prior model's learned representations) when the current dataset is too small to learn good embeddings from scratch — transfer from a larger related task.
- For tree-based models (XGBoost/LightGBM), embeddings are less standard — target encoding with smoothing, or native categorical support (LightGBM's `categorical_feature`, which does optimal split-finding on categories directly), is usually preferred over forcing embeddings into a GBM pipeline.

### 1.4 Other Encodings Worth Knowing
- **Frequency encoding**: replace category with its count/frequency in training data — simple, leakage-free (doesn't touch the target), useful as a complementary feature even alongside target encoding.
- **Ordinal encoding**: only appropriate when categories have a genuine order (e.g. `low/medium/high`); using it on nominal categories imposes a fake numeric ordering that misleads linear models (tree models are more robust to this but it's still not recommended).
- **Hashing trick**: hash category strings into a fixed number of buckets — bounds dimensionality for extremely high-cardinality or streaming/unknown-vocabulary categoricals, at the cost of collisions.

## 2. Time-Series Feature Patterns

Extends the skill's `add_time_features` (cyclical dow/month encodings). The higher-risk, higher-value features are lags and rolling windows — and they are also the easiest place to leak the future into training.

### 2.1 Lag Features

```python
df["y_lag_1"]  = df.groupby("entity_id")["y"].shift(1)
df["y_lag_7"]  = df.groupby("entity_id")["y"].shift(7)
df["y_lag_28"] = df.groupby("entity_id")["y"].shift(28)   # captures weekly seasonality at 4-week granularity
```

Choose lags that match known seasonality (1 = yesterday, 7 = same weekday last week, 28/364 = same period last month/year) rather than an arbitrary grid — domain knowledge about the business cycle should drive lag selection before you let feature-selection tooling pick from hundreds of candidate lags.

### 2.2 Rolling Window Aggregates

```python
df["y_roll_mean_7"] = (
    df.groupby("entity_id")["y"]
      .shift(1)                     # CRITICAL: shift before rolling
      .rolling(window=7, min_periods=3)
      .mean()
)
df["y_roll_std_7"] = (
    df.groupby("entity_id")["y"].shift(1).rolling(window=7, min_periods=3).std()
)
```

The `.shift(1)` before `.rolling()` is the single most common leakage bug in time-series feature engineering: `rolling(7).mean()` **without** the shift includes the current row's own `y` value inside its own rolling window average — the model is trained on a feature that partially contains the label it's predicting. This inflates offline validation metrics dramatically and then fails in production, where the current-period label obviously isn't available yet at prediction time.

### 2.3 Leakage From Future Data — the General Pattern

Time-series leakage takes several forms beyond the rolling-window bug:

- **Global statistics computed over the full timeline** (e.g. `df["y"].mean()` used to fill NaNs, or a global min-max scaler fit on train+test together) leak future distribution information backward into earlier rows.
- **Cross-sectional joins that aren't time-aware**: joining a "customer lifetime value" feature computed as of *today* onto historical training rows — the feature didn't exist yet at the historical prediction time (this is the most common real-world leakage bug in production feature stores, often called "feature time-travel").
- **Train/test splits that shuffle rather than split by time.** For time series, always use a forward-chaining / time-series split (train on `t < T`, test on `t >= T`), never `train_test_split(shuffle=True)`.

```python
from sklearn.model_selection import TimeSeriesSplit

tscv = TimeSeriesSplit(n_splits=5)
for train_idx, test_idx in tscv.split(X):
    # train_idx always precedes test_idx in time
    ...
```

- **Point-in-time correctness discipline**: every feature must be computable using *only* information available at (or before) the timestamp of the prediction it's paired with. When building a feature store or backfilling historical training data, tag each feature with an "as-of" timestamp and join with an explicit `feature_timestamp <= label_timestamp` condition — this is the production-grade fix for the join-leakage case above.

### 2.4 Additional Time-Series Feature Families

- **Expanding-window statistics** (`.expanding()` instead of `.rolling(window=k)`): cumulative mean/count up to `t-1`, useful when the full history (not just a fixed window) is informative and window size is otherwise an arbitrary hyperparameter.
- **Time-since-event features**: days since last purchase, days since account creation — strong signal for churn/lifecycle models, and computed correctly by construction (naturally point-in-time) as long as "last purchase" only looks backward from the prediction timestamp.
- **Calendar/holiday features**: is_holiday, days_to_next_holiday, is_month_end — combine with the skill's cyclical `sin`/`cos` day-of-week/month encodings for models sensitive to periodicity (linear models, neural nets); tree-based models can often use raw integer day-of-week/month directly since they split on thresholds rather than assuming linear/circular structure.
- **Cross-entity aggregation features** (careful with leakage here too): e.g. "average conversion rate across all products in this category, excluding this product" — same leave-one-out logic as target encoding, and same requirement to compute using only same-fold/past-time data.

## 3. Feature Selection Methods

Extends the skill's model-evaluation workflow (SHAP validation, overfit_gap check). Feature selection should have a concrete, checkable criterion — not "remove features that seem unimportant."

### 3.1 Filter Methods (model-agnostic, fast, run before any model)

- **Correlation with target**: Pearson (linear) or Spearman (monotonic, robust to outliers) correlation; drop features below a threshold (e.g. `|corr| < 0.01`) *only* as a first-pass screen — this misses features that matter only in interaction or nonlinearly, so don't use it as the sole filter.
- **Mutual information**: `sklearn.feature_selection.mutual_info_classif` / `mutual_info_regression` — captures nonlinear dependence that correlation misses, at the cost of needing more data to estimate reliably.
- **Variance threshold**: drop near-constant features (`VarianceThreshold(threshold=0.0)` removes zero-variance columns outright) — cheap, catches broken/constant pipeline outputs.
- **Multicollinearity via VIF**: Variance Inflation Factor `VIF_j = 1 / (1 - R_j^2)` where `R_j^2` comes from regressing feature `j` on all other features. `VIF > 10` (some practitioners use 5) flags a feature that's largely redundant given the others — matters for linear/logistic models' coefficient stability and interpretability, matters much less for tree ensembles' predictive performance.

### 3.2 Wrapper Methods (model-in-the-loop, more expensive, capture interactions)

- **Recursive Feature Elimination (RFE)**: fit model, rank features by importance/coefficient magnitude, drop the weakest, refit; repeat until target feature count reached. `sklearn.feature_selection.RFECV` adds cross-validation to pick the feature count automatically rather than guessing it.
- **Forward/backward stepwise selection**: greedily add (forward) or remove (backward) the feature that most improves/least hurts a CV score. Computationally expensive for large feature sets (`O(p^2)` model fits) but interpretable and directly optimizes the actual evaluation metric.

### 3.3 Embedded Methods (selection happens during model fitting)

- **L1 (Lasso) regularization**: drives coefficients of uninformative features exactly to zero; the regularization strength `alpha` controls sparsity — pick via CV on a held-out fold, not a fixed default.
- **Tree-based importance**: `feature_importances_` (mean decrease in impurity) is fast but biased toward high-cardinality numeric features; prefer **permutation importance** (shuffle one feature, measure the drop in held-out score) or **SHAP values** (already in the skill's model-evaluation checklist) for a more trustworthy ranking, since both are computed on held-out data and reflect actual predictive contribution rather than in-sample split-purity artifacts.

```python
from sklearn.inspection import permutation_importance

result = permutation_importance(model, X_val, y_val, n_repeats=10, random_state=42, scoring="roc_auc")
importances = result.importances_mean  # negative/near-zero => feature not useful on held-out data
```

### 3.4 A Concrete Selection Criterion (avoiding "remove what seems unimportant")

1. Compute permutation importance (or SHAP) on a **held-out validation fold**, not train.
2. Rank features; drop any feature whose importance confidence interval (from `n_repeats` permutation runs) includes zero or is negative — it is not reliably contributing.
3. Re-run cross-validation with the reduced feature set; confirm the mean CV score does not drop beyond a pre-agreed tolerance (e.g. `-0.005 AUC`) and that `overfit_gap` (train − test score, per the skill's model-evaluation checklist) improves or stays flat — a shrinking overfit gap after removing low-importance features is a good sign you removed noise, not signal.
4. For features removed on multicollinearity grounds (high VIF) rather than low importance, prefer keeping the one with clearer business interpretability or lower missingness/instability over time, since predictive power alone won't distinguish between two highly correlated candidates.
5. Document *why* each dropped feature was removed (below-threshold importance CI, high VIF against feature X, leakage risk) — feature selection decisions get revisited every time the model is retrained, and an undocumented removal gets silently reversed or re-litigated.

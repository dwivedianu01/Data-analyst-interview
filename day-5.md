# Day 05 - Real Project Scenarios (Data Analyst / Data Science)

Practical, on-the-job situations that actually come up during projects — not textbook definitions. Each includes a model answer and some include hands-on code.

---

# Part A: Data Analyst Scenarios

## A1. Two dashboards show different numbers for "revenue"

**Scenario:** Finance's dashboard shows $2.1M for last month, your dashboard shows $1.9M. Same company, same month. Leadership asks why.

**How to approach:**
- Don't guess — reconcile at the row level, not the total. Pull both queries' row counts and filters side by side.
- Common root causes: different **definition** of revenue (gross vs net of refunds/tax), different **date basis** (order date vs ship date vs payment date), different **status filters** (includes cancelled/pending orders or not), timezone cutoff differences on "last month," or one source including test/internal accounts.
- Fix: agree on a single **metric definition** documented in a shared data dictionary/semantic layer (e.g., dbt metrics or a governed view), and have both dashboards read from the same certified table instead of each team writing its own SQL.

---

## A2. A stakeholder asks for "all the data" with no clear question

**Scenario:** A VP says "just give me a dashboard of everything about customer churn."

**How to approach:**
- Push back constructively: ask what **decision** this will drive (e.g., "are we deciding where to invest retention budget, or diagnosing a recent spike?"). The decision determines the right metrics and grain.
- Propose a short scoping conversation: who's the audience, what time grain, what segments matter, what does "churn" mean here (cancelled subscription? inactive 90 days? non-renewal?).
- Ship a focused v1 (3-5 key metrics) rather than a kitchen-sink dashboard — easier to validate and iterate.

---

## A3. Report numbers look right but a key metric trend suddenly flattens

**Scenario:** Daily active users has been flat at exactly the same number for 5 days, which is statistically implausible.

**How to approach:**
- Suspect a **pipeline failure silently serving stale data** (e.g., a failed job that didn't alert, and the dashboard is reading yesterday's cached/last-successful table).
- Check load timestamps / `_loaded_at` columns on the source table, check job/task run history for silent failures or a task that's suspended.
- Lesson: dashboards should show a **"data as of" timestamp**, and pipelines should have freshness monitors that catch "flat/stale" patterns, not just hard failures.

---

## A4. A metric definition changes mid-quarter

**Scenario:** Product redefines "active user" (used to be "logged in," now "performed a key action"). Historical trend lines will show a discontinuity.

**How to approach:**
- Never silently overwrite history — recompute the metric **both ways** for a transition period and clearly annotate the dashboard with a vertical marker/footnote at the change date.
- Store the metric **definition/version** alongside the data (e.g., a `metric_version` column) so historical comparisons stay honest.
- Communicate the change to consumers before it ships, not after someone notices a sudden "drop."

---

## A5. Ad-hoc request conflicts with a deadline for a recurring report

**Scenario:** You have a board report due in 2 hours, and your manager's boss asks for an urgent one-off pull.

**How to approach:**
- Triage explicitly rather than silently picking one: confirm priority with whoever can arbitrate (usually your direct manager), state the trade-off ("I can do the urgent pull now and the board report will be ~30 min late, or vice versa").
- This is a communication/prioritization skill question as much as a technical one — interviewers are checking you won't just silently blow a deadline.

---

## A6. Sampling bias in a customer survey analysis

**Scenario:** An NPS survey shows 90% satisfaction, but support tickets and churn are both rising.

**How to approach:**
- Suspect **non-response/survivorship bias** — unhappy customers who already churned aren't in the active-customer survey pool, and dissatisfied active customers may be less likely to respond at all.
- Check response rate and compare respondent demographics/segment mix to the full customer base; weight or caveat results accordingly.
- Cross-validate the survey signal against **behavioral data** (tickets, churn, usage decline) rather than trusting a single self-reported metric in isolation.

---

## A7. Outliers are skewing an average

**Scenario:** "Average order value" jumped 40% this week. One enterprise client placed a single $500K order.

**How to approach:**
- Check for and separately report **median** alongside mean, and consider segment-level reporting (e.g., exclude or flag enterprise/outlier accounts above a threshold, or report B2B vs B2C separately).
- Don't silently drop the outlier if it's a legitimate transaction — it's real revenue — but the *narrative* ("average customer is spending more") would be misleading without context.
- Use percentile breakdowns (p50/p90/p99) in recurring reports so single large transactions don't distort the headline number.

---

## A8. Data looks complete but a join is silently dropping rows

**Scenario:** A report's total order count is consistently ~3% lower than the source system's count.

**How to approach:**
- Suspect an **inner join dropping unmatched rows** — e.g., joining orders to customers and losing orders where the customer record is missing/deleted, or a type mismatch (string vs int key, leading zeros, trimmed whitespace) causing join misses.
- Validate with `LEFT JOIN` + `IS NULL` check on the join key to quantify exactly how many rows fail to match and why, before deciding whether to fix the join, fix the data, or just document the known gap.

---

## A9. Business wants a forecast, but you only have 3 months of history

**Scenario:** Leadership wants a 12-month revenue forecast for a product launched 3 months ago.

**How to approach:**
- Be upfront about the limitation — 3 months isn't enough to detect seasonality and a simple trend-line extrapolation will likely be wrong for a 12-month horizon.
- Offer alternatives: use a comparable/analogous product's early trajectory as a proxy, build a range (best/expected/worst case) instead of a single number, and commit to revisiting the forecast monthly as more data arrives rather than promising one static number.

---

## A10. Row-level access control requested mid-project

**Scenario:** Regional sales managers should only see their own region's data in a shared dashboard, but the dashboard currently queries one shared table.

**How to approach:**
- Implement **row-level security** at the data layer (e.g., Snowflake row access policies, or a view filtered by a mapping table of user → region) rather than relying on the BI tool to hide rows, which can be bypassed via export/API.
- Test with least-privilege accounts, not just the admin account, before rollout.

---

# Part B: Data Science Scenarios

## B1. A/B test shows a "winning" variant, but the lift disappears after launch

**Scenario:** A checkout redesign showed +5% conversion in a 2-week A/B test; after full rollout, conversion is flat.

**How to approach:**
- Common causes: **novelty effect** (users reacted to change, not better design), **test duration too short** to capture a full business cycle (weekday/weekend mix), **sample ratio mismatch** or **peeking** (stopping early on a significant-looking result), or the test population wasn't representative of the full rollout population (e.g., excluded mobile users).
- Always check sample ratio mismatch and pre-registration of the stopping rule/duration before trusting a result; prefer a **holdback group** post-launch to validate the effect persists.

---

## B2. Model performs great offline, poor in production

**Scenario:** A churn prediction model has 90% offline AUC but business says predictions "don't match reality."

**How to approach:**
- Check for **data leakage** in training (e.g., a feature only known *after* churn, like "cancellation date," accidentally included, or target leakage through an ID join).
- Check **training/serving skew** — are production features computed the same way as training features (same time windows, same joins, same null-handling)?
- Check for **temporal leakage**: was the train/test split random instead of time-based, letting the model "see the future" relative to how it'll be used in production?

---

## B3. Feature distribution drifts over time

**Scenario:** A fraud model's precision degrades month over month after a successful launch.

**How to approach:**
- Monitor **feature drift** (e.g., population stability index / distribution comparisons between training data and live traffic) and **label drift** (fraud patterns evolve as fraudsters adapt).
- Set up automated drift alerts and a retraining cadence/trigger rather than a "set and forget" model; keep a rolling evaluation set with ground truth to track real-world precision/recall over time, not just at launch.

---

## B4. Missing data isn't random

**Scenario:** A customer income field is missing for 30% of rows, and it's missing disproportionately for younger customers.

**How to approach:**
- This is **Missing Not At Random (MNAR)** — naive mean imputation will bias the model by shifting the "missing" group toward the overall mean incorrectly.
- Options: add a **missingness indicator flag** as its own feature (often the missingness itself is predictive), model-based imputation conditioned on related features (e.g., age, occupation), or treat missing as its own category for tree-based models.
- Always investigate *why* it's missing (a specific signup flow that doesn't collect income?) before choosing a technique.

---

## B5. Class imbalance in a fraud/rare-event model

**Scenario:** Fraud is 0.3% of transactions. A model achieves 99.7% accuracy by predicting "not fraud" every time.

**How to approach:**
- Accuracy is the wrong metric here — use **precision/recall, PR-AUC, or F1** at a chosen operating threshold, and frame cost explicitly (cost of a missed fraud vs cost of a false positive blocking a legitimate transaction).
- Techniques: class weighting, resampling (SMOTE/undersampling) *applied only to training data, never to the validation/test set*, or threshold tuning on the precision-recall curve to match business cost tradeoffs.

---

## B6. Correlation vs causation in a business recommendation

**Scenario:** Analysis shows customers who use feature X churn less. Product wants to force all users into feature X to reduce churn.

**How to approach:**
- Flag the **confounding risk**: engaged customers are both more likely to discover feature X *and* more likely to stay, regardless of X itself (reverse causation / confounding by engagement level).
- Recommend a **randomized experiment** (A/B test forcing exposure to feature X) before committing engineering resources to a causal claim based on observational correlation alone.

---

## B7. Stakeholder wants a single accuracy number for a complex model

**Scenario:** Leadership asks "is the model good? Just give me one number."

**How to approach:**
- Translate the technical metric into **business impact** instead of a raw score: e.g., "at the current threshold, we catch 80% of fraud while incorrectly flagging 2% of good transactions, which costs $X in manual review but saves $Y in fraud losses."
- If one number is truly needed, pick the one tied to the business decision (e.g., expected dollar impact), not an academic metric like raw accuracy/AUC that doesn't map to a decision.

---

## B8. Retraining cadence trade-off

**Scenario:** A recommendation model could be retrained daily, weekly, or monthly. How do you decide?

**How to approach:**
- Weigh: how fast the underlying patterns actually change (catalog turnover, seasonality), the cost/time to retrain and validate, risk of deploying a bad model (need a rollback/shadow-deployment safety net), and whether there's enough new labeled data between intervals to meaningfully change the model.
- A common middle ground: retrain on a schedule (e.g., weekly) but monitor drift continuously and trigger an **out-of-cycle retrain** if drift/performance crosses a threshold, rather than a purely fixed cadence.

---

## B9. Explaining a "black box" model to a regulator or business owner

**Scenario:** A credit-risk model needs to justify individual denial decisions.

**How to approach:**
- Use **model-agnostic explainability** (SHAP/LIME) to produce per-decision feature attributions, and maintain a simpler, interpretable **challenger model** (e.g., logistic regression) to sanity-check that the complex model's behavior is directionally reasonable.
- Document which features are used, confirm none are **legally protected proxies** (e.g., zip code correlating with race), and keep an audit trail of model versions tied to decisions for compliance.

---

## B10. Experiment is contaminated by network effects

**Scenario:** A marketplace discount experiment randomizes by individual buyer, but buyers interact with the same limited pool of sellers, so the control group is indirectly affected (sellers run out of discounted inventory faster).

**How to approach:**
- This is **SUTVA violation** (Stable Unit Treatment Value Assumption) / interference between units — individual-level randomization isn't valid when units share a constrained resource.
- Fix: randomize at a higher, less-interacting level (e.g., by **market/region** or **seller cluster** instead of by individual buyer — a cluster-randomized / switchback design) to reduce cross-contamination.

---

# Part C: Hands-on Practical Exercises

## C1. Reconcile two "revenue" sources in pandas

Given two DataFrames representing the same month from two systems, write code to find exactly which `order_id`s differ and why (amount mismatch vs missing from one side).

<details>
<summary>Reference answer</summary>

```python
import pandas as pd

merged = finance_df.merge(
    analytics_df, on="order_id", how="outer",
    suffixes=("_finance", "_analytics"), indicator=True
)

missing_in_analytics = merged[merged["_merge"] == "left_only"]
missing_in_finance = merged[merged["_merge"] == "right_only"]
amount_mismatch = merged[
    (merged["_merge"] == "both") &
    (merged["amount_finance"] != merged["amount_analytics"])
]
```
This isolates the three failure modes separately instead of just reporting "totals don't match."
</details>

---

## C2. Detect a flat/stale metric automatically

Write a function that flags if a daily metric series has been suspiciously constant (a likely sign of a stale pipeline) for `n` consecutive days.

<details>
<summary>Reference answer</summary>

```python
import pandas as pd

def is_stale(series: pd.Series, n: int = 5) -> bool:
    return series.tail(n).nunique() == 1

# Example
daily_dau = pd.Series([10234, 10198, 10401, 10401, 10401, 10401, 10401])
print(is_stale(daily_dau, n=5))  # True -> alert
```
In production, pair this with a freshness check (`MAX(loaded_at)`) to distinguish "value is genuinely flat" from "pipeline didn't refresh."
</details>

---

## C3. Missingness-aware imputation

Given a DataFrame `df` with a numeric `income` column that is MNAR (missing non-randomly), add a missingness indicator and impute using a group-conditional median instead of a global mean.

<details>
<summary>Reference answer</summary>

```python
df["income_missing"] = df["income"].isna().astype(int)
df["income"] = df.groupby("age_group")["income"].transform(
    lambda s: s.fillna(s.median())
)
```
The `income_missing` flag preserves the signal that missingness itself may carry, instead of erasing it through imputation.
</details>

---

## C4. Sample ratio mismatch (SRM) check for an A/B test

Given `n_control` and `n_treatment` counts and an intended 50/50 split, write a quick statistical check to flag an SRM before trusting the test's results.

<details>
<summary>Reference answer</summary>

```python
from scipy.stats import chisquare

def check_srm(n_control: int, n_treatment: int, expected_ratio=(0.5, 0.5), alpha=0.01) -> bool:
    total = n_control + n_treatment
    expected = [total * r for r in expected_ratio]
    _, p_value = chisquare([n_control, n_treatment], f_exp=expected)
    return p_value < alpha  # True => SRM detected, don't trust the results yet

print(check_srm(49500, 50500))
```
A significant SRM means something (bucketing bug, bot filtering, logging gap) skewed assignment, and the experiment's conclusions should not be trusted until it's fixed.
</details>

---

## C5. Precision/recall at a business-chosen threshold

Given predicted fraud probabilities and true labels, find the probability threshold that achieves at least 80% recall while maximizing precision.

<details>
<summary>Reference answer</summary>

```python
import numpy as np
from sklearn.metrics import precision_recall_curve

precision, recall, thresholds = precision_recall_curve(y_true, y_proba)

valid = recall[:-1] >= 0.80
best_idx = np.argmax(precision[:-1][valid]) if valid.any() else None
best_threshold = thresholds[valid][best_idx] if best_idx is not None else None
print(best_threshold, precision[:-1][valid][best_idx])
```
Tying the threshold choice to a business constraint (minimum recall) instead of an arbitrary 0.5 cutoff reflects how fraud/risk thresholds are actually chosen in production.
</details>

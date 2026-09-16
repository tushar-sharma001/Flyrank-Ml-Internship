# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Tushar
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/tushar-sharma001/Flyrank-Ml-Internship
- **Date:** 2026-09-17

## 0. Abstract

Out of a large content inventory, only a small fraction of pages can realistically be reviewed by a human editor in a given week. This project builds a ranked scoring system on FlyRank's pseudonymized search & engagement warehouse to answer which pages should be reviewed first. A transparent rule-based baseline (`stale_but_visible`) was compared against Logistic Regression and Random Forest on an identical client-grouped test split. An early label definition produced a 98.2% positive rate that turned out to be mostly day-to-day noise; a corrected label (smoothed 11-day future window, 10% magnitude threshold, minimum-volume floor) brought this to a defensible 37%, at which point Random Forest reached an AUC of 0.937 against the rule baseline's 0.626 — a large, genuine gap. The output is a decision-support ranking for content reviewers, not a causal claim about what refreshing a page will do.

## 1. Problem Framing

**Decision supported:** out of thousands of pages with search visibility, which ones should a content reviewer look at first this week.

**Unit of analysis:** one (client, content item) pair, features aggregated over the 90 days prior to a decision point.

**Output:** a ranked score plus one reason code (`stale_but_visible` for the baseline; a continuous probability for the model).

**Action a human takes:** manually review the top-ranked candidates and decide to refresh, expand, protect, prune, or monitor.

**Cost of a wrong call:** a false positive wastes limited reviewer time on a page that didn't need it; a false negative lets a real decline compound silently until it's a much larger loss. Because reviewer time is the scarce resource, this project optimizes for precision at the top of the ranked list, not recall across the whole inventory.

**Why ML helps here:** no single observable signal separates decliners from non-decliners cleanly (the strongest safe candidate feature correlated with the label at only ~0.11 in early checks) — the real pattern, if it exists, is spread thinly across several weakly-informative signals at once, which is exactly the case where a model combining signals can outperform a single-threshold rule.

## 2. Data Safety

**Data used:** FlyRank's pseudonymized internship warehouse (`flyrank_pseudonymized_warehouse_release_v20260703`), tables `dim_content` and `fact_content_daily_performance`, month `2026-03` for development with label windows extending into `2026-04`. The sealed final month was never touched for label logic.

**Deliberately excluded, and why:**
- `health_score`, `priority_score`, `action_type` — FlyRank's own product-decision flags. Not shipped in the release on purpose, and using them would mean the model learns to copy an existing rule rather than discover real signal.
- Raw query, URL, and title text — not shipped; scrambled before release, never reconstructed.
- GA4 rows before a client's `ga4_data_start` — zero-filled placeholders, not real zero engagement, so excluded rather than treated as a true zero.

**Leakage risks considered:**
- `trend_direction` / `trend_pct` were never used as features — they are the source of the decline label itself, and using them as inputs too would mean training the model to predict its own ingredients.
- `client_hash_id`, `content_hash_id`, `url_hash_id`, `keyword_hash_id` are used only for joins, grouping, and the leakage check itself — never as model features.
- Explicit leakage checks run and passed: zero client overlap between train and test (verified by set intersection, not assumed); zero banned/product-flag columns present in the final feature list; feature-importance sanity check against a 0.7 dominance threshold (see Section 6 for the one flag this surfaced).

**Client-identifying content:** none present anywhere in `work/` — all IDs are pseudonymous hashes used for grouping only.

## 3. Baseline

**Rule:** `stale_but_visible` — flag a page if `days_since_update >= 180` AND `impressions_prior90 >= 500`, scored by `impressions_prior90` when both conditions hold.

**Why it's a fair comparison:** it is built from the exact same prior-90-day features available to the model, on the exact same test split, so any gap between it and the model reflects real added value, not a difference in data access.

**Baseline numbers (client-grouped test split, corrected label):** Precision@50 = 0.860, AUC = 0.626.

## 4. Model / Analysis

**Method:** Logistic Regression first, then Random Forest — chosen because the target is a genuine yes/no observed outcome. Logistic Regression keeps the first model fully interpretable (a coefficient can be pointed to directly); Random Forest was added second only to test whether added complexity earns its place over both the baseline and the linear model.

**Feature list:** `impressions_prior90`, `clicks_prior90`, `avg_position`, `ctr_prior90`, `days_since_update` — all aggregated strictly over the prior-90-day window, so nothing was knowable only after the decision point.

**Deliberately left out:** raw GA4 channel splits (`sessions_ai`, `sessions_paid`, etc., individually) — rolled into aggregate signals rather than treated as five separate sparse features; AI-session data specifically is too sparse to support its own feature per the lane guide's own warning (30K rows out of 78M in the full warehouse).

**Target/proxy, one sentence:** a binary flag for whether a content item's smoothed future-window average impressions (days 25–35 out) fall more than 10% below its prior-90-day average impressions, with a minimum 10-impression daily floor to exclude pages too small to meaningfully "decline."

## 5. Evaluation

**Split:** 80/20 client-grouped (37 training clients / 144,011 rows, 10 held-out test clients / 32,726 rows) — whole clients held out, never split across train/test, because pages from the same client likely share structural patterns a random row split could let the model exploit.

**Split honesty check:** a direct before/after comparison against a naive random row split (under the earlier label) showed nearly identical AUC (0.774 naive vs. 0.773 grouped) — reassuring evidence the model wasn't inflating its score by recognizing clients.

**Metrics, model vs. baseline, same split:**

| Method | Precision@50 | AUC |
|---|---|---|
| Base rate (random) | 0.370 | 0.500 |
| Rule baseline | 0.860 | 0.626 |
| Logistic Regression | 0.860 | 0.755 |
| Random Forest | 0.860 | 0.937 |

**Base rate vs. base rate framing:** the test cohort's base rate (majority class = "not declining") is 63%; all three ranking methods clearly beat the 0.370 random-selection precision@50, so the precision@50 tie across methods is not simply an artifact of a high base rate — it reflects that the top-50 candidates genuinely overlap across methods, while AUC (base-rate independent) shows Random Forest separates the full ranking far better than the others.

**Error analysis:** the three worst errors are all false negatives — pages the model rated ~1% decline probability that genuinely declined. All three share a specific profile: real impression volume (200–232), zero clicks, and exactly 34 days since last update. The model appears to read "recently updated" as an implicit safety signal even when zero engagement over that same window is arguably the stronger warning sign.

## 6. Interpretation

**Feature importances (Random Forest):** `impressions_prior90` 71.3%, `ctr_prior90` 15.3%, `clicks_prior90` 10.3%, `days_since_update` 2.2%, `avg_position` 0.9%.

**Plain-words read:** raw visibility volume and click-through behavior dominate; staleness and position barely matter once volume and CTR are accounted for. This is a genuine surprise relative to an earlier (buggy-label) run, where `days_since_update` was the dominant feature at 63.7% — that earlier result is now understood to be an artifact of label noise, not a durable signal, and is reported here as a negative result worth knowing, not hidden.

**One flag surfaced and investigated, not hidden:** `impressions_prior90`'s 71.3% importance crosses the 0.7 self-check threshold. Judged not to be literal leakage (the feature is legitimately prior-window, never touches the future), but a real, named confound: the label compares each page against its own prior average, so high-volume pages mechanically have more room to show a 10%+ drop, independent of any real, actionable trend.

## 7. Recommendation

A three-tier review priority, built directly from the error analysis above:

1. **Priority 1 — high impressions, zero clicks, position outside top 20.** This is the model's specific, named blind spot; review these manually even when the model's own score is low.
2. **Priority 2 — top Random Forest scores confirmed by the rule baseline.** Where both independently-built signals agree, confidence is highest.
3. **Priority 3 — high Random Forest score where the rule baseline disagrees.** Worth a second look to understand why the simple rule missed it.

**Confidence and limits, stated plainly:** this is decision-support, not a guarantee about any individual page. It has not been tested outside one development month's data, and it does not claim that refreshing a flagged page will cause recovery — that requires a controlled experiment this dataset cannot provide.

## 8. Reproducibility

**Commands to re-run from a fresh clone:**
```bash
git clone https://github.com/tushar-sharma001/Flyrank-Ml-Internship.git
cd Flyrank-Ml-Internship
pip install duckdb huggingface_hub scikit-learn pandas numpy --break-system-packages
# open work/notebooks/w05_model.ipynb in Colab or Jupyter, set HF_TOKEN, Run All
```

**Random seeds:** `np.random.seed(42)` for the client-grouped train/test split; `random_state=42` for both `LogisticRegression` and `RandomForestClassifier`.

**Environment:** `duckdb`, `huggingface_hub`, `scikit-learn`, `pandas`, `numpy` — installed fresh per notebook run via `%pip install`, no pinned lockfile beyond what's in each notebook's setup cell.

**Sealed/holdout evaluation status:** the final warehouse month is sealed and has not yet been evaluated against — no claim of a completed blind holdout run is made in this report. The client-grouped split used throughout is a development-time honesty check, not the sealed evaluation.

## 9. Acknowledgments & Data Credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

> **Claims checklist confirmed:** observed / measured / directional / decision-support language used throughout · no causal claims made without an experiment · no claim of predicting Google's algorithm · no client-identifying details anywhere in this report · numbers above match the notebook outputs from the corrected-label re-run (w05_model.ipynb, w06_validation_audit.ipynb).

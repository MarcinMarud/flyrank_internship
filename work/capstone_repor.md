# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** [Marcin Marud]
- **Lane:** Refresh / Content Opportunity Scoring (Lane 2), with a Lane 1 signal-audit foundation
- **Repo:** [https://github.com/MarcinMarud/flyrank_internship]
- **Date:** [14.08.2026]

## 0. Abstract

This project studies which observable content and search signals predict whether a page's visibility declines the following month, using FlyRank's warehouse release. Features were built from March 2026 (impressions, clicks, position, word count, staleness), and the target was defined from a genuinely future window — April 2026 impressions dropping below 70% of March impressions — on the 60,744 content items with meaningful March visibility. A transparent staleness-and-visibility baseline rule was compared against Logistic Regression and Random Forest models on a client-grouped holdout split. Logistic Regression reached the best result, an ROC AUC of 0.595 and average precision of 0.426 against a 33.3% test-set base rate, a real but modest lift over the 0.459 AUC baseline rule. The output is intended as a decision-support ranking for content reviewers, not proof that any signal causes future decline.

## 1. Problem framing

**Decision supported:** which pages a content strategist should review first, out of a much larger inventory than review capacity allows.

**Unit of analysis:** one content item (`content_hash_id`), with features aggregated over March 2026.

**Output:** a ranked queue with a numeric score, one reason code (`stale_visible_page`), and an action label.

**Action a human takes:** a strategist works down the ranked list until their review capacity for the cycle is used up.

**Cost of a wrong call:**
- False positive (flagged, no real April decline) — wasted reviewer time.
- False negative (missed, real April decline) — a real problem goes unaddressed and compounds.

**Why ML over a fixed rule:** a fixed rule combines a small number of hand-chosen signals at fixed thresholds. A model can weigh several signals jointly, and this project tests whether that joint weighting produces a real, non-circular lift over the rule.

## 2. Data safety

**Data used:** `fact_content_daily_performance` for two months — March 2026 (`month=2026-03`, feature window) and April 2026 (`month=2026-04`, target window) — joined to `dim_content` for static content attributes, from the FlyRank internship warehouse on Hugging Face.

**Feature window:** March 2026, 331,437 content-client rows after cleaning. **Target window:** April 2026, 362,172 content-level rows, used only to compute the future outcome, never as a model feature. After restricting to content with meaningful March visibility (impressions at or above the 50th percentile, 242), 60,744 content items remained eligible for labeling.

**Deliberately excluded:** product-computed fields (`health_score`, `priority_score`, `action_type`) — not present in this release, and would be circular if they were. No raw client names, domains, URLs, or private queries appear anywhere in this data or in this report.

**Leakage risks considered and addressed:**
- `content_updated_date` was capped at the end of the March feature window before computing staleness, so no page's "days since update" reflects an update that happens after the feature window closes.
- The target is computed strictly from April data, and no April field is present among the model's features (`impressions_90d`, `clicks_90d`, `avg_position`, `word_count`, `days_since_update` — all March-only).
- Pseudonymous IDs (`content_hash_id`, `client_hash_id`) are used only for joining and grouping, never as features.

## 3. Baseline

**Rule:** a page is flagged `stale_visible_page` if its March `days_since_update` is at or above the 70th percentile and its March `impressions_90d` is at or above the 60th percentile (242). 24,304 of 60,744 eligible rows were flagged.

**Why it's a fair comparison:** the rule uses only March features, the same features and the same client-grouped test split as the models, and is scored against the same April-derived target.

**Baseline result on the held-out test split:** ROC AUC 0.459, average precision 0.322, against a 33.3% test-set positive rate. The AUC sitting just under 0.5 indicates the rule's rank ordering carries almost no discriminative signal for next-month decline on its own — a legitimate, reportable result, not a data error.

## 4. Model / analysis

**Method:** Logistic Regression as the interpretable comparison point; Random Forest (`class_weight="balanced"`) as the complexity test.

**Features:** `impressions_90d`, `clicks_90d`, `avg_position`, `word_count`, `days_since_update` — all from the March feature window only, each knowable at the decision point (same-window observed measurements or static content attributes).

**Target/proxy, one sentence:** `target = 1` if a content item's April impressions fell below 70% of its March impressions, restricted to items with above-median March visibility — a genuinely future-observed outcome, built entirely from a month the features never touch.

## 5. Evaluation

**Split:** grouped by `client_hash_id` (70/30), confirmed zero client overlap between train and test. Train: 20,898 positives; test: 2,271 positives.

**Results, same split, same metric:**

| Method | ROC AUC | Average Precision |
|---|---:|---:|
| Baseline rule | 0.459 | 0.322 |
| Logistic Regression | 0.595 | 0.426 |
| Random Forest | 0.577 | 0.407 |

**Base rate:** 33.3% positive in the test set. Both models clear the base rate on average precision (0.426 and 0.407 vs. 0.333), and both clear the baseline rule's AUC by a meaningful margin (0.595 and 0.577 vs. 0.459). Logistic Regression is the best-performing method on both metrics in this run.

**Error analysis:** the Logistic Regression / Random Forest comparison used to size errors shows 114 false positives (flagged, no real April decline) and 871 false negatives (missed, real April decline occurred) on the Random Forest at the 0.7 / 0.3 probability thresholds used. The heavier weight on false negatives suggests the model is currently conservative — it under-flags true decliners more often than it over-flags stable pages, which is worth factoring into how a reviewer sets their own confidence threshold.

## 6. Interpretation

**Feature importance (Random Forest):**

| Feature | Importance |
|---|---:|
| `word_count` | 0.300 |
| `impressions_90d` | 0.292 |
| `avg_position` | 0.290 |
| `clicks_90d` | 0.106 |
| `days_since_update` | 0.012 |

Importance is spread fairly evenly across word count, visibility, and position, with staleness contributing very little — a different picture from an earlier same-window version of this analysis, where staleness and visibility dominated because the label had been built directly from those two features. With a genuine future-window target, no single feature dominates, which is consistent with a non-circular result.

**Signal audit:**
- Staleness buckets now cover all 60,744 eligible rows (60,674 / 59 / 11 across `<90d`, `90-180d`, `180-365d`). The two older buckets have very small n (59 and 11), so their median values are not reliable on their own. Verdict: **MIXED** — the `<90d` bucket dominates the data and shows no clear direction against the sparser older buckets; this signal needs a larger or differently-binned sample before a confident verdict.
- CTR by position tier: median CTR decreases from 0.0028 (tier 1-3) to 0.0024 (4-10) to 0.0017 (11-20), before ticking down further to 0.0007 (21+) — a broadly monotonic decline consistent with the CTR-fix logic. Verdict: **CONFIRMED**.

**Surprise / negative result:** the baseline rule's AUC (0.459) sits at essentially chance level for predicting next-month decline, even though it uses the same signals a model finds more use in when combined. This is a legitimate finding: staleness and visibility, combined by fixed threshold alone, aren't enough — the lift over baseline comes from the model's weighting, not from the underlying features being individually powerful.

## 7. Recommendation

Use the ranked queue as a starting point for review, not a final verdict. The strongest, most consistent pattern among top-ranked candidates combines high March visibility (impressions in the tens of thousands) with 120+ days since the last recorded update — a page that is both seen and aging. A strategist should treat a high score as a hypothesis to check, since patterns like recent redirects, content consolidation, or metadata-only edits (not reflected as a "real" update) can still produce a high score without an actual content problem.

Given the current false-negative rate is higher than the false-positive rate, a reviewer relying on this queue should treat "not flagged" with some caution rather than as reassurance — the model is more likely to miss a real decliner than to raise a false alarm.

**Confidence:** moderate on the Logistic Regression result (real, non-circular lift over baseline); low on any claim beyond decision-support ranking.

## 8. Reproducibility

Re-run from a fresh clone:
```
git clone https://github.com/MarcinMarud/flyrank_internship.git
cd flyrank_internship
```
Notebooks, in order:
```
work/notebooks/w02_research_question.ipynb
work/notebooks/w02_ml_task_framing.ipynb
work/notebooks/w03_data_contract.ipynb
work/notebooks/w04_baseline_score.ipynb
work/notebooks/w05_model.ipynb
work/notebooks/capstone.ipynb
```
Random seed: `random_state=42` used throughout (`GroupShuffleSplit`, `RandomForestClassifier`).

Environment: DuckDB + httpfs, pandas, scikit-learn, matplotlib, via Colab's default Python environment plus `HF_TOKEN` stored as a Colab Secret.

No sealed/holdout claim is made — the client-grouped test split is a standard holdout, described as such.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai

---

> **Claims checklist:** observed / measured / directional / decision-support language throughout; no causal claims, no claims about Google's algorithm, no client-identifying details.
> **Metrics vs. base rate:** base rate is 33.3% positive (test set); both models' average precision (0.426, 0.407) exceeds this, reported alongside it above rather than standalone.
> Numbers above match the notebook output for this run — re-run to confirm before submitting.

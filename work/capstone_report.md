# Prioritizing Content Refresh Opportunities with Search, Engagement, and Freshness Signals

**Author:** Dev  
**Lane:** Lane 2 — Refresh / Content Opportunity Scoring  
**Project:** FlyRank ML Internship  
**Repo:** https://github.com/Dev-hashh/flyrank-ml-internship-starter

## Abstract

Content teams need a practical way to prioritize which pages should be reviewed when the number of candidate pages exceeds available editorial capacity. This study asks whether observable search, engagement, content, and freshness signals can rank pages for earlier review. We compare a transparent Week-4 baseline with Logistic Regression, Decision Tree, and Random Forest experiments, and use client-grouped validation to test the Random Forest on seven unseen clients. On that grouped holdout, the Random Forest achieved ROC-AUC 0.844, Average Precision 0.854, and Precision@50 0.980, compared with 0.498, 0.482, and 0.320 for the baseline on the same test rows. These results support directional decision support for review prioritization, not a causal claim that refreshing a ranked page will improve traffic, rankings, CTR, or engagement.

## 1. Introduction / Problem Statement

The decision supported by this project is: **which pages should be reviewed first for a possible content action?**

The unit of analysis is one content page per row. The output is a ranked refresh-priority queue with a score, suggested action, reason codes, and confidence label. A human editor can use the queue to decide which pages deserve earlier investigation.

The cost of a wrong call is asymmetric. Prioritizing a page that does not need intervention consumes editorial capacity; missing a genuinely important declining page can delay investigation of a potentially meaningful opportunity. ML is useful here because the inventory contains many observable search, engagement, content-depth, and freshness signals that can be combined into a ranking rather than relying on one rule.

The system is explicitly a **review-prioritization tool**. It does not decide that a page is "bad", does not automatically rewrite or delete content, and does not establish that a refresh will cause recovery.

## 2. Data

### Dataset and release

The internship guide identifies Lane 2 as the Refresh / Content Opportunity Scoring lane and permits either the warehouse release or the starter dataset. The warehouse release is `flyrank_pseudonymized_warehouse_release_v20260703`. Its documented tables include `dim_clients` (104 rows), `dim_content` (519,606 rows), `fact_content_daily_performance` (78,835,655 daily rows), and `fact_content_query_90d` (2,414,248 client-content-query-hash rows).

The experiments reported in this paper were developed and validated on the **30,000-row anonymized starter dataset**, not on the full 78.8M-row daily warehouse. This distinction is important: the warehouse provides scale and structure, while the reported model metrics are earned on the starter slice.

### Windows and grain

The starter feature vector contains 90-day aggregate search and engagement measures. The decline proxy is based on `trend_direction`, which is calculated from a 30-day versus previous-30-day impression comparison. One row represents one pseudonymized content page.

### Included signals

The model used observable numeric and categorical signals including:

- search volume, competition, and CPC;
- word count and character count;
- 90-day impressions, clicks, pageviews, sessions, users, engaged sessions, AI sessions, and scroll events;
- days with impressions and sessions;
- last-30-day and previous-30-day impressions, clicks, and sessions;
- content age and days since last update;
- CTR, average position, engagement rate, scroll rate, and AI traffic percentage;
- competition, content type, intent, age, freshness, word-count, character-count, impression, and position tiers.

### Exclusions and safety

The model deliberately excluded `content_id` and `client_id` as features. They are used for joining and grouping, not as meaningful predictive signals.

It also excluded:

- `trend_direction` — it directly defines the label;
- `trend_pct` — it is part of the same trend calculation and can reveal the label;
- the Week-4 baseline score;
- baseline/reason-code outputs and final refresh outputs.

No client names, domains, URLs, private queries, credentials, or raw page text are used in the public paper.

## 3. Methodology

### Target / proxy label

The target is:

`is_declining_label = 1 when trend_direction == "down"; otherwise 0`

This is a **proxy label for observed decline**, not a future refresh-success label.

The overall positive rate in the 30,000-row modeling table was 0.542. In the client-grouped test set, the positive rate was 0.511.

### Baseline

The project first established a transparent rule-based refresh score using visibility, freshness risk, position opportunity, and content-depth gap signals. The baseline was evaluated on the same held-out rows as the learned model.

For the final grouped validation comparison, the Week-4 baseline achieved:

- ROC-AUC: 0.498
- Average Precision: 0.482
- Precision@50: 0.320

### Models

The Week-5 experiment compared:

- Logistic Regression
- Decision Tree
- Random Forest

The custom Week-5 comparison produced:

| Model | ROC-AUC | Average Precision | Precision@50 |
|---|---:|---:|---:|
| Logistic Regression | 0.848 | 0.866 | 1.000 |
| Decision Tree | 0.876 | 0.859 | 1.000 |
| Random Forest | 0.844 | 0.854 | 0.980 |

The downstream validation audit and action-playbook pipeline used the Random Forest, so the paper's main validation result is the Random Forest under the stricter client-group holdout. This keeps the reported queue, validation audit, and action playbook aligned.

### Validation design

The final validation used `GroupShuffleSplit` with 20% of clients held out and `random_state=42`.

- Training: 23,837 rows from 25 clients
- Test: 6,163 rows from 7 clients
- Client overlap: 0
- Training positive rate: 0.550
- Test positive rate: 0.511

This is preferable to a random row split for the intended cross-client generalization question because pages from the same client can share structural characteristics.

### Leakage audit

The audit asked whether a field was available at decision time, directly derived from the target, an output of another decision system, or capable of making the test artificially easy. IDs were retained only for grouping/joining. Trend fields and baseline outputs were excluded from model features.

## 4. Results

### Main model vs baseline — same client-group holdout

| Metric | Week-4 Baseline | Random Forest |
|---|---:|---:|
| ROC-AUC | 0.498 | 0.844 |
| Average Precision | 0.482 | 0.854 |
| Precision@50 | 0.320 | 0.980 |

The baseline positive rate in the grouped test set was 51.1%. Precision@50 should therefore be interpreted alongside that base rate rather than in isolation.

At the top 50 positions, Precision@50 of 0.980 corresponds to 49 of the first 50 ranked pages matching the positive proxy label in this evaluated test set. This is a measured ranking result, not evidence that those pages would necessarily benefit from a refresh.

### Random split vs client-group holdout

The validation audit also measured the same Random Forest under a random row split:

| Validation design | ROC-AUC | Average Precision | Precision@50 |
|---|---:|---:|---:|
| Random row split | 0.913 | 0.928 | 1.000 |
| Client-group holdout | 0.844 | 0.854 | 0.980 |

The random split had 31 clients represented in both train and test. The grouped split had no client overlap. The lower grouped performance is therefore the more relevant measurement for cross-client generalization in this project.

### Error analysis

At a 0.50 probability threshold on the grouped test set, the audited Random Forest produced:

- Precision: 0.749
- Recall: 0.766
- Predicted positives: 3,223
- Actual positives: 3,149
- False positives: 810
- False negatives: 736

False positives included pages with high model probability but a negative proxy label; these are candidates for human review because the model may be responding to patterns associated with decline without the label being positive. False negatives included several very low-volume pages, illustrating that pages with sparse evidence can be difficult to prioritize reliably.

### Model interpretation

The Random Forest's reported descriptive feature-importance ordering in the starter pipeline was led by:

1. `days_with_impressions`
2. `log_impressions_90d`
3. `avg_position`
4. `content_age_days`
5. `char_count`
6. `word_count`
7. `log_clicks_90d`
8. `ctr`
9. `scroll_rate`
10. `days_with_sessions`

These are model-behavior indicators, not causal effects.

## 5. Limitations & Honest Framing

1. **Proxy target.** The label describes observed decline in the measured window. It does not represent whether a future refresh will succeed.
2. **Starter-slice scope.** The reported experiments use 30,000 anonymized rows. They are not a benchmark on the full 78.8M-row warehouse.
3. **Observational data.** The analysis does not establish that refreshing a page causes recovery.
4. **Generalization.** Seven unseen clients provide a stricter cross-client test, but they do not guarantee performance on every future client or page type.
5. **Metric interpretation.** Precision@50 is a ranking metric for the review queue. It should be read alongside the 51.1% test-set base rate.
6. **Feature importance.** Importance values describe model behavior within the experiment; they do not show that changing a feature will cause the outcome to change.
7. **Human context.** The model cannot observe the full editorial, technical, legal, competitive, or business context of a page.
8. **Data freshness.** Search demand, rankings, competitors, seasonality, and measurement coverage can change after feature collection.

The safe claim is therefore: **the model provides directional decision support for prioritizing pages for human review.**

## 6. Ranked Recommendations / Action Playbook

The final action queue contains 30,000 pages and maps ranked evidence to review paths.

| Rank | Action | Share | Human review focus |
|---|---|---:|---|
| 1 | Monitor | 43.61% | Collect more evidence before substantive change |
| 2 | Refresh | 27.29% | Confirm persistent decline and content freshness |
| 3 | Refresh + review CTR | 22.18% | Inspect CTR relative to visibility and title/snippet alignment |
| 4 | Refresh + review engagement | 6.64% | Check usefulness, completeness, and engagement context |
| 5 | Expand + refresh | 0.27% | Confirm meaningful coverage gaps; avoid filler |

The queue is a **prioritized review list**, not an automatic editorial command.

### Recommended workflow

**Model ranking → human evidence review → content decision → approved action → follow-up measurement**

For a high-ranked page, a reviewer should check:

- whether the decline is persistent;
- whether the page still matches search intent;
- whether the content is outdated or incomplete;
- whether CTR is being interpreted relative to search position;
- whether seasonality, SERP changes, consolidation, or low volume provide another explanation;
- whether a concrete content improvement exists.

The no-go list includes automatic publishing, deleting, merging/redirecting, metadata changes, or claiming that a refresh caused recovery.

### Monitoring / retraining triggers

Future use should monitor input data quality, score/action distributions, confidence distributions, reason-code frequencies, and performance on newly labeled data. A trigger should lead to **detect → investigate → revalidate → retrain only if justified → compare with the existing model → update the playbook**.

## 7. Reproducibility

The core workflow is:

```bash
python scripts/run_all.py
```

The repository's reference pipeline prepares features, builds the baseline, trains the models, evaluates the ranked queue, and builds the report.

Key artifacts used by this paper include:

- `work/notebooks/w04_baseline_score.ipynb`
- `work/notebooks/w05_model.ipynb`
- `work/notebooks/w06_validation_audit.ipynb`
- `work/notebooks/w07_action_playbook.ipynb`
- `work/notebooks/capstone.ipynb`
- `work/outputs/action_queue.csv`
- `work/outputs/action_summary.csv`
- `work/outputs/playbook_summary.json`
- `work/figures/action_distribution.png`

The modeling notebooks use `random_state=42`. The client-group holdout uses a 20% test fraction. The environment should be recreated from the repository's `requirements.txt`.

## 8. Acknowledgments & Data Credit

Built on the **FlyRank ML Internship dataset**.

Data credit: [FlyRank](https://flyrank.ai)

This work uses the anonymized internship dataset and follows the project's public-safety rules. No client names, domains, URLs, private queries, credentials, or raw page text are included in this public artifact.

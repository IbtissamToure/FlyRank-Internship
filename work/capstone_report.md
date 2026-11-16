# Capstone Report — Freestyle: leakage-safe decline-risk ranking

- **Author:** Ibtissam Toure
- **Lane:** Freestyle — growth / momentum prediction (next-month decline risk)
- **Repo:** https://github.com/IbtissamToure/FlyRank-Internship
- **Paper:** https://ibtissamtoure.github.io/FlyRank-Internship/
- **Date:** November 2026

Every number below is produced by `work/notebooks/capstone.ipynb` and recorded in
`work/outputs/capstone_metrics.json` (seed 42).

## 0. Abstract

Content teams can only review a handful of pages per client each month, so the question is which pages to look at before they lose search visibility. Using 23,727 anonymized pages from 31 client sites in the FlyRank ML Internship dataset, I split each page's 90-day history into three 30-day windows and learned only from days 31–90 to rank the risk of a ≥20% impressions drop in the most recent 30 days. A logistic-regression ranking, validated with 5-fold cross-validation grouped by client, reached ROC-AUC 0.60 against 0.53 for an "already falling" rule and 0.51 for a "highest visibility" rule, and its top-50 review list per test fold caught a decline 84% of the time against a 63% base rate; it also out-ranked the falling-traffic rule inside 16 of the 20 larger clients. The output is a reason-coded, per-client review playbook — decision support for prioritising human review, not a forecast of any single page and not a model of Google's ranking system.

## 1. Problem framing

- **Decision:** which pages of a client site a human should review first this month.
- **Unit of analysis:** one content page (eligible: ≥ 10 impressions in days 31–60).
- **Output:** a risk score, ranked within each client, mapped to Protect / Improve / Investigate / Monitor with reason codes.
- **Cost of a wrong call:** false positive = one wasted review; false negative = a ≥20% visibility loss noticed a month late.
- **Why ML:** both editor rules are near chance in this data (AUC 0.51 and 0.53); combining volume, click capture, momentum, age and content context orders the queue better.

## 2. Data safety

- Data: the anonymized starter release (`data/raw/content_refresh_anonymized.csv`, 30,000 pages, 32 clients). The warehouse (March 2026 partition) was used for the Week-3 data contract only.
- Windows: W3 = days 61–90 (`*_90d − *_last_30d − *_prev_30d`, never negative), W2 = days 31–60 (`*_prev_30d`), W1 = days 1–30 (`*_last_30d`, label only).
- Excluded: `trend_direction`, `trend_pct` (the label); all 90-day aggregates and tiers (contain W1); `days_since_last_update` (68% of pages updated inside W1); `provider_used`, `model_used` (dictionary: not a feature); `content_id`, `client_id` (grouping only). A forbidden-column assertion runs in the notebook.
- Nothing client-identifying appears in `work/` or `docs/`; clients are shown as "client 01…".

## 3. Baseline

| Method | ROC-AUC | Precision@50 | Precision@200 |
|---|---|---|---|
| Base rate (no ranking) | 0.500 | 0.632 | 0.632 |
| Visibility rule (rank by W2 impressions — Week-4 rule, leakage-safe) | 0.505 | 0.504 | 0.539 |
| Already-falling rule (rank by W3→W2 drop — Week-2 rule) | 0.533 | 0.588 | 0.612 |

Same pages, same folds, same metrics as the model.

## 4. Model / analysis

- **Method:** logistic regression (primary), histogram gradient boosting (check). Seed 42.
- **Features (19):** log impressions/clicks/sessions in W2 and W3; W3→W2 momentum for each; W2 CTR; content age at T; log search volume, competition, CPC; log word count; `has_keyword_data`, `has_word_count`; content type; main intent.
- **Target:** `impressions_last_30d < 0.8 × impressions_prev_30d` (reproduces `trend_direction == "down"` on all rows). Base rate 61.6%.

## 5. Evaluation

- **Split:** `GroupKFold(n_splits=5)` by client, out-of-fold scores; zero client overlap asserted.

| Method | ROC-AUC (sd) | Avg precision | Precision@50 | Precision@200 |
|---|---|---|---|---|
| Logistic regression | **0.597** (0.019) | **0.712** | **0.836** | **0.812** |
| Gradient boosting | 0.586 (0.031) | 0.698 | 0.732 | 0.766 |

- **Within-client** (20 clients with ≥ 100 pages): median AUC 0.60 vs 0.53 (falling rule) and 0.45 (visibility rule); median Precision@20 0.80 vs 0.60 / 0.55 (median base rate 0.67). LR beats the falling rule in 16/20 clients and is above chance in 19/20.
- **Robustness:** single 80/20 client split (seed 42) AUC 0.63, P@50 0.92 vs 0.54; floor ≥ 100 W2 impressions AUC 0.63 vs 0.57.
- **Leakage experiment:** honest 0.597 → + 90-day aggregates 0.687 → + `trend_pct` 1.000.
- **Week-5 feature set on the forward label:** AUC 0.552, P@50 0.756.
- **Errors:** four clients where the rule ranks as well or better; gains are modest (AUC 0.60), so single-page judgements are unreliable.

## 6. Interpretation

- Younger pages decline more (72% vs 54% for pages under ~6 months at T).
- Pages that already earn clicks are steadier; impressions without clicks look fragile.
- Feed-sourced articles and unknown-intent pages carry higher risk; comparison articles lower ("unknown intent" may partly proxy site differences).
- Negative result: momentum alone is weak — already-falling pages declined 65% vs 58%; recent spikes did not reliably revert.

## 7. Recommendation

| Action | Rule | Pages | Observed decline |
|---|---|---|---|
| 1 Protect — review now | top 10% risk in client, ≥ 1,000 W2 impressions | 750 | 79% |
| 2 Improve — snippet / intent review | top 10%, zero W2 clicks | 1,476 | 74% |
| 3 Investigate — refresh or merge | other top 10% | 163 | 72% |
| 4 Monitor | top 10–25% | 3,561 | 70% |
| 5 No action | rest | 17,777 | 58% |

Human review is required before any change; never automate publish / rewrite / delete / redirect. Monitor Precision@20 per client, base rate, feature drift and reason-code mix monthly; retrain or retire if precision falls to the base rate two months in a row. Confidence: directional — one forward month only.

## 8. Reproducibility

```bash
git clone https://github.com/IbtissamToure/FlyRank-Internship
cd FlyRank-Internship && pip install -r requirements.txt
cd work/notebooks && jupyter nbconvert --to notebook --execute capstone.ipynb
```

Seed 42; scikit-learn ≥ 1.4 (run with 1.9.1), pandas ≥ 2.2. The notebook writes `work/outputs/capstone_metrics.json` (committed), `work/outputs/capstone_action_queue.csv` (gitignored) and the four figures in `docs/img/`.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

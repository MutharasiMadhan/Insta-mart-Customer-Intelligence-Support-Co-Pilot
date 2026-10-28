# Insta-mart-Customer-Intelligence-Support-Co-Pilot
Instamart Customer Intelligence &amp; Support Copilot — Capstone Project

# Instamart Customer Intelligence — Part A

Reproducible cleaning, SQLite storage, SQL analysis and pandas exploration for the synthetic Instamart capstone data. All money values are in INR. The supplied deterministic generator is retained as `generate_data.py`; its raw outputs are written under `data/raw/` and policy/scenario files are also generated for later parts.

## 1. Reproduce

Run these commands from the repository root using Python 3.10 or newer:

```bash
git switch -c main
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1
# macOS/Linux instead: source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
pip freeze > requirements.txt
python generate_data.py
python audit_data.py
python clean_data.py
python validate_data.py
python run_queries.py
python analysis.py
```

The generator is deterministic (seed 42) and creates `data/raw/customers.csv`, `products.csv`, `orders.csv`, `tickets.csv`, and other capstone inputs. It also writes policy documents under `data/policies/`. Raw generated exports are not edited by the cleaning pipeline. Commit the repository and scripts on branch `main`; do not commit `.venv/`.

## 2. Raw-data audit

`audit_data.py` reports shape, dtypes, missing-value count and percentage for each raw table/column. It uses `keep_default_na=False` for tickets so blank ticket text remains an empty string, and also reports blank strings separately. The specific raw audit figures are:

| Audit item | Result |
|---|---:|
| Raw order rows | 13,084 |
| Exact duplicate order rows | 119 |
| Rows with `pending` status | 656 |
| Distinct raw `order_id` | 12,309 |
| Unparseable raw `order_amount` | 120 |
| Negative raw `delivery_minutes` | 106 |
| Blank raw `delivery_minutes` | 2,541 |
| Blank customer ages | 61 |
| Customer ages outside 10–100 | 6 |
| Distinct raw city spellings | 24 |
| Distinct nonblank raw gender spellings | 10 |
| Blank genders | 30 |

## 3. Documented cleaning rules

### Customers

- `city`: trim surrounding whitespace and apply title case. This standardizes 476 of 1,200 raw city values; six city names remain. Final counts: Pune 227, Delhi 210, Bengaluru 197, Chennai 191, Hyderabad 191 and Mumbai 184.
- `gender`: trim and lowercase before mapping `male`/`m` to `male`, `female`/`f` to `female`; blank or unrecognized spellings become `unknown`. 950 values are standardized; final counts are female 610, male 560, unknown 30.
- `age`: convert to numeric, set the 6 values outside 10–100 to missing, and fill all 67 missing/invalid ages (61 blank plus 6 out of range) with the median age for that standardized city using `groupby('city').transform('median')`. The final data has no missing ages and ages range from 18 to 60.
- `signup_date`: parse as datetime and serialize to `YYYY-MM-DD` in the clean CSV/database.

### Orders

Rules are applied in the required order:

1. Parse `order_ts` as datetime. The cleaned database stores it as text in `YYYY-MM-DD HH:MM:SS` format.
2. Drop exact duplicate rows: 119 rows removed (13,084 → 12,965).
3. Sort by `order_id` and timestamp, then retain the latest row per order. This removes 656 older duplicate order records and leaves 12,309 unique orders: 11,110 completed and 1,199 cancelled, with no pending rows.
4. Convert `order_amount` to numeric; the latest-record dataset has 112 unparseable/missing amounts. Fill those using `round(price_inr * quantity * (1 - discount_pct / 100), 2)` from the product catalogue. Final missing amount count is zero.
5. Convert delivery times to numeric and set negative values to missing (105 negative durations remain after deduplication). For completed orders, fill 669 missing delivery values using the completed-order median for the matching product category via `groupby('category').transform('median')`. Cancelled orders retain missing delivery times: all 1,199 cancelled rows have a missing `delivery_minutes` because they were never delivered.

Products retain the supplied 40-row catalogue, with `price_inr` converted to numeric. Clean CSVs are written to `data/clean/`; SQLite tables `customers`, `products`, and `orders` are written to `data/instamart.db`.

## 4. Validation

`validate_data.py` asserts more than eight required reference checkpoints, including the final city and gender counts, age completeness, one row per order, final status counts, amount completeness, delivery-time missingness rules, product count, SQLite tables and completed revenue. It prints exactly `ALL CHECKPOINTS PASSED` only when all assertions pass.

Expected completed-order revenue is **INR 2,998,668.80**.

## 5. SQL analysis

Seven labelled queries are stored in `sql/queries.sql` and run by `run_queries.py` through `pandas.read_sql_query` against `data/instamart.db`.

- Q1: 30 customers.
- Q2: 30 city/category cells, total INR 2,998,668.80; largest cell Bengaluru / Electronics, INR 233,909.45.
- Q3: 11 products priced above the catalogue average.
- Q4: 25 customers with no completed order.
- Q5: 550 customers above average completed spend; average customer spend is INR 2,552.06 (rounded to two decimals).
- Q6: ranks completed spend within city using `RANK`, `DENSE_RANK`, and `ROW_NUMBER`. The top customers per city match the assignment references. Ties share the same `RANK`, so later rank values can skip positions while `ROW_NUMBER` always assigns a distinct sequential number.
- Q7: 183 daily rows; the final cumulative completed revenue is INR 2,998,668.80.

## 6. pandas cross-check and exploration

`analysis.py` recreates Q2 with `pd.pivot_table(..., margins=True)`, prints the city-level named aggregations, completed-order distribution statistics, Tukey 1.5×IQR fences and outlier counts, and the correlation matrix for `order_amount`, `discount_pct`, `quantity`, and `delivery_minutes`.

The completed order amount mean is INR 269.91 versus a median of INR 135.00, with skewness about 4.61. Q1 is INR 64.80 and Q3 is INR 297.00; the upper fence is INR 645.30, with 897 values above it and none below. The largest absolute correlation is between `order_amount` and `discount_pct` (about −0.095), slightly larger in absolute value than the correlation between `order_amount` and `quantity` (about +0.087). Correlation of this size does not establish causation.

# Part B — Customer Segments and Lapse Prediction

## RFM checks

At snapshot `2025-10-01 00:00:00`, there are 1,175 customers with completed orders. Recency median/min/max is 19/0/180 days; frequency mean/max is 9.46/33; monetary mean/max is INR 2,552.06/INR 14,766.55.

## K-means choice

| K | Inertia | Silhouette |
|---:|---:|---:|
| 2 | 1820.8 | 0.465 |
| 3 | 1268.2 | 0.445 |
| 4 | 743.1 | 0.526 |
| 5 | 617.4 | 0.480 |
| 6 | 503.2 | 0.482 |
| 7 | 408.5 | 0.416 |
| 8 | 366.5 | 0.412 |

K=4 is selected because it has the highest silhouette score and the K=3 to K=4 inertia reduction is substantial while four segments remain actionable. Inertia always decreases with more clusters because each observation can be assigned to a closer centroid; the lowest inertia alone rewards complexity and does not tell us whether the segmentation is stable or useful to a campaign. PCA explained-variance ratios: 0.634 and 0.218, total 0.852.

## Segment profiles

See `segment_profiles.csv` for the exact profiles and recommended actions. Segment IDs are arbitrary labels, not ordered ranks. The Premium flag is assigned only to the segment with the highest mean monetary value.

## Lapse cohort and baseline

The cutoff is `2025-09-01`. Cohort: 1,156 customers; lapsed: 443 (38.32%); train/test: 924/232. The always-not-lapsed baseline has accuracy 0.616 and lapsed precision/recall/F1 of 0/0/0. No order dated on or after the cutoff enters model features, and Task 3 segment labels are not used as features. All encoders and scalers are fitted on training rows only; oversampling is also restricted to the training rows.

## Tuned models

| Model | Best parameters | Best CV F1 | Test confusion matrix (TN, FP / FN, TP) | Lapsed precision | Lapsed recall | Lapsed F1 |
|---|---|---:|---|---:|---:|---:|
| Logistic regression | C=100 | 0.651 | [[111, 32], [31, 58]] | 0.644 | 0.652 | 0.648 |
| Random forest | max_depth=8, min_samples_leaf=10 | 0.683 | [[114, 29], [31, 58]] | 0.667 | 0.652 | 0.659 |

The tuned random forest is selected because it has the stronger test F1 and precision while matching logistic regression's recall. Missing a lapsing customer risks losing future revenue, but wrongly targeting a loyal customer wastes campaign budget and may train customers to wait for discounts; the balance should be monitored after deployment.

## Oversampling experiment

Training-only oversampling balances the training classes at 570 each. Logistic regression before oversampling: precision 0.644, recall 0.652, F1 0.648. After oversampling: precision 0.589, recall 0.854, F1 0.697. Recall changes most, finding more potential lapsers while lowering precision and increasing the likely number of unnecessary contacts.

## Interpretation and artifacts

- Random-forest leading features: frequency, monetary, recency, average delivery minutes, and cancellation rate. See `random_forest_top_features.csv` for importances.
- Logistic coefficient signs and values are in `logistic_coefficient_signs.csv`. Positive means higher lapse log-odds, negative means lower lapse log-odds, conditional on other features.
- The selected pipeline is saved to `lapse_model.joblib`; reloaded test predictions match exactly.
- The top ten Premium-segment test customers by predicted lapse probability are in `retention_list.csv`; 2 of the 10 were actually lapsed in this run.

**Limitation:** The retention list is illustrative and not a leakage-free evaluation because its Premium flag comes from whole-window RFM, which includes the September label window; the model's own features remain pre-cutoff only.

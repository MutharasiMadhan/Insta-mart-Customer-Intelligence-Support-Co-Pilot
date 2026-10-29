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

## Part B — customer segments and lapse prediction

Run `python part_b.py` from the project root after installing `requirements.txt`. The script consumes only `data/clean/orders.csv`, `data/clean/customers.csv`, and the clean product catalogue, and writes its deliverables to `outputs/`.

### RFM and segmentation

The snapshot is `2025-10-01 00:00:00`; the RFM table contains 1,175 customers with at least one completed order. The run reproduced recency median/min/max of 19/0/180 days, frequency mean/max of 9.46/33, and monetary mean/max of INR 2,552.06/INR 14,766.55. K-means uses a `StandardScaler` fitted to the RFM matrix and `KMeans(n_init=10, random_state=42)` for each K from 2 through 8. Full inertia and silhouette results are saved in `outputs/kmeans_metrics.csv`.

K=4 is selected: its silhouette score (0.526) is the best of the tested K values and its inertia drops substantially from K=3 (1268.2) to K=4 (743.1), while four profiles are still straightforward to turn into distinct campaigns. Inertia always decreases as K increases because adding clusters can only make it easier to fit observations closer to their assigned centroids; K=8 therefore has lower inertia than K=2, but that does not account for model complexity or whether the resulting segments are useful to the business. K=4 segment profiles, names and recommended actions are saved in `outputs/segment_profiles.csv`; assignments are in `outputs/customer_segments.csv`. The highest-mean-monetary segment is flagged Premium. **Cluster IDs are arbitrary labels, not a ranking or an inherent order.** PCA is fit to the scaled RFM matrix; the two explained-variance ratios are approximately 0.634 and 0.218 (total 0.852).

### Lapse definition, split and leakage controls

The cohort contains every customer with at least one completed order before `2025-09-01`. A customer is labelled lapsed when they have no completed order from `2025-09-01` through `2025-09-30` inclusive. This run has 1,156 cohort customers, 443 lapsed (38.32%), and a stratified 924/232 train/test split (`random_state=42`). Every behaviour feature is built using orders strictly before `2025-09-01`; no order on or after the cutoff enters the features, and the whole-window segment labels from Task 3 are not model features. Numeric scaling and categorical one-hot encoding (`drop='first'`) are fitted inside scikit-learn pipelines on training rows only and applied to the test rows; no encoder, scaler or resampler saw test rows. Feature engineering checks that no feature column contains a missing value. The training-only scaler mean for frequency is printed alongside the full-cohort mean to demonstrate why they can differ.

### Model results from the reproducible run

- Always predicting not lapsed: test accuracy 0.616; lapsed precision 0.000, recall 0.000, F1 0.000 (`zero_division=0`).
- Logistic regression grid: `C ∈ {0.01, 0.1, 1, 10, 100}`, `max_iter=1000`, 5-fold CV with F1 scoring. Best C was 100 (best CV F1 0.651); test confusion matrix `[[111, 32], [31, 58]]`, lapsed precision 0.644, recall 0.652, F1 0.648.
- Random forest grid: 100 trees, seed 42; `max_depth ∈ {3, 5, 8}`, `min_samples_leaf ∈ {5, 10, 20}`, 5-fold CV with F1 scoring (45 fits). Best parameters were depth 8 and leaf size 10 (best CV F1 0.683); test confusion matrix `[[114, 29], [31, 58]]`, lapsed precision 0.667, recall 0.652, F1 0.659.
- The selected model is the tuned random forest. Its lapsed-class F1 clears the 0.55 target. Missing a customer who is about to lapse can mean losing future orders, so recall matters; however, false positives also spend discount budget on customers who would have ordered anyway. The random forest gives a slightly stronger F1 and precision than logistic regression on the held-out set while matching its recall, making it the more balanced choice for a first retention campaign.
- Training-only oversampling duplicates lapsed training rows with replacement to balance the training set at 570/570. Logistic regression changed from precision/recall/F1 of 0.644/0.652/0.648 to 0.589/0.854/0.697 on the unchanged test set. Recall moved most: the campaign can identify more at-risk customers, at the cost of contacting more loyal customers (lower precision), so campaign capacity and offer cost should guide the threshold.

The top five random-forest features and three largest-magnitude numeric logistic coefficients are saved to `outputs/random_forest_top_features.csv` and `outputs/logistic_coefficient_signs.csv`. In this run, frequency, monetary, recency, average delivery time, and cancellation rate were the five leading forest features. The displayed logistic coefficient sign is conditional on the other model features: a positive sign means higher lapse log-odds, and a negative sign means lower lapse log-odds; these are associations, not causal effects. The selected model is saved to `outputs/lapse_model.joblib`, then reloaded and checked to reproduce every original test prediction exactly. The top ten Premium-segment customers from the test set by predicted lapse probability are written to `outputs/retention_list.csv`; in this run, 2 of the 10 were actually lapsed.

### Limitations and Next Steps

The retention list is illustrative and not a leakage-free evaluation because its Premium flag comes from whole-window RFM that includes the September label window, although the model features themselves use only pre-cutoff orders. Validate on a later time-based holdout before campaign deployment, monitor precision/recall and offer cost, and calibrate a contact threshold to retention-team capacity.

## Part C — support ticket triage with guardrails

Run `python part_c.py` after Parts A and B. The pipeline processes `data/raw/tickets.csv` in file order and reads the segment names and Premium flags from `outputs/customer_segments.csv`. It writes `outputs/triage_results.csv` (few-shot, 100,000-token budget), `outputs/triage_budget_demo.csv` (zero-shot, 6,000-token budget), `outputs/zero_shot_confusion.csv`, `outputs/part_c_summary.json`, and `outputs/part_c_report.md`.

### Input and privacy guardrails

- Blank or whitespace-only text is marked `skipped_empty`; text with `est_tokens(text) > 400` is marked `flagged_long`. Neither reaches the model boundary. The four long tickets estimate to 734 tokens each.
- Redaction runs in the required order: email ` [\\w.+-]+@[\\w-]+\\.[\\w.]+ ` → `[EMAIL]`; phone ` \\b[6-9]\\d{9}\\b ` → `[PHONE]`; order ID `ORD-\\d{6}` → `[ORDER_ID]`. Across all 164 tickets, the script counts 55 email, 91 phone, and 132 order-ID substitutions (278 total), then asserts no pattern remains. It also checks the exact user content passed to the mock boundary for residual matches. The mock-only `ticket_id` argument is metadata, not part of the user message; the user content is exactly the redacted ticket text.
- Zero-shot and few-shot prompts both include `CLASSIFY`, the four allowed categories, and the required JSON shape. Only the few-shot prompt contains `EXAMPLES:` and labelled examples including fraud.
- Retry logic allows an initial call plus at most three retries, uses exponential backoff with jitter, and prints one log line per retry. Each full run calls `mock_llm.reset()` immediately before processing. In each full run, the 19 transient errors recover on retry.
- Valid JSON is accepted only when `category` is one of the four allowed values. Otherwise, for non-JSON text only, the pipeline uses `re.findall(r"[a-z]+", text.lower())` and accepts exactly one allowed category word. Invalid JSON/category results and all guardrail/budget outcomes route to `human_review`; the pipeline never guesses a category.

### Results and token budget

The full zero-shot and few-shot runs each classify 147 tickets (135 JSON plus 12 keyword fallback); 9 invalid outputs, 4 empty tickets, and 4 long tickets do not receive an automated category. Zero-shot accuracy is 85.7% (126/147) and few-shot accuracy is 100% (147/147). In this local run, zero-shot usage is 13,144 total tokens or 89.4 tokens per classified ticket; few-shot usage is 25,000 total tokens or 170.1 per classified ticket. Few-shot therefore costs about 80.7 extra tokens per classified ticket (11,856 tokens across the run) and removes 21 classification errors among the classified tickets. That trade-off is worthwhile for sensitive or costly misroutes, especially fraud, while production deployments should compare the incremental token cost with human-review and misrouting costs.

The zero-shot budget demonstration uses a 6,000-token budget and records 84 `budget_exhausted` tickets. The budget is checked before each model-call sequence; the successful response that crosses the threshold is included in usage, and later calls are stopped once the running total is at least the budget. `triage_results.csv` has 164 rows and columns `ticket_id, customer_id, status, category, segment_name, priority, route, tokens_used`. Routing order is human review for empty/long/invalid/budget-exhausted, security team for fraud, premium desk for Premium customers otherwise, and support queue for the remainder.

**Mock interface:** `mock_llm.py` uses the supplied offline chat interface, `response = chat(messages, tools=None)`, and its response fields (`choices[0].message.content` and `usage.total_tokens`). The triage pipeline passes the redacted ticket text as the entire user-message content; ticket IDs are retained only in local audit metadata and are not appended to the prompt. The mock is deterministic, offline test scaffolding, not a real hosted model client.

### Limitations and Next Steps

Only rule-validated categories (`classified_json` and the exactly-one-category `classified_keyword` fallback) are treated as automated classifications; invalid outputs and skipped/long/budget-exhausted tickets go to human review. The mock's accuracy is designed to match the assignment reference and does not establish real-world model performance. Before deployment, test with the original provided mock and then a real model under approved privacy controls, monitor category-level errors and retries, and audit the human-review queue. The Part B Premium flag is a post-hoc whole-window label, so any retention list based on it is illustrative and not a leakage-free evaluation.

## Guardrails and privacy

Tickets: 164; empty skipped: 4; long flagged: 4.
Redactions across all 164 ticket texts: email=55, phone=91, order_id=132, total=278; residual matches after redaction: 0.
The code asserts that no email, phone, or order-ID pattern remains in any redacted text before a model call. The mock-only ticket ID is metadata and is not part of the user message content.

## Zero-shot vs few-shot

| Metric | Zero-shot | Few-shot |
|---|---:|---:|
| Classified tickets | 147 | 147 |
| Accuracy on classified tickets | 85.7% | 100.0% |
| Total response tokens | 13144 | 25000 |
| Tokens per classified ticket | 89.4 | 170.1 |

### Zero-shot confusion table

| true_category   |   billing |   delivery |   technical |   fraud |
|:----------------|----------:|-----------:|------------:|--------:|
| billing         |        38 |          0 |           0 |       0 |
| delivery        |         0 |         36 |           0 |       0 |
| technical       |        10 |          0 |          29 |       0 |
| fraud           |        11 |          0 |           0 |      23 |

Few-shot uses 11856 additional tokens (80.7 more per classified ticket). In this mock run, it improves classified-ticket accuracy from 85.7% to 100.0%; that is worth paying for when the cost of misrouting a fraud or other support case exceeds the small incremental token cost. If production volume or token prices make that difference material, reserve few-shot prompting for ambiguous or high-risk tickets and monitor accuracy by category.

## Retry and status checks

Zero-shot retries logged: 19; few-shot retries logged: 19; each full run resets the mock first. Status counts in the few-shot full run: `{"classified_json": 135, "classified_keyword": 12, "flagged_invalid_output": 9, "flagged_long": 4, "skipped_empty": 4}`.
Budget demo used 6000 tokens as its stopping threshold and ended with 84 budget_exhausted tickets.

## Routing and human review

`triage_results.csv` contains 164 rows. Empty, long, invalid-output, and budget-exhausted tickets route to human_review; fraud routes to security_team at HIGH priority; Premium customers with otherwise valid categories route to premium_desk at HIGH; remaining tickets route to support_queue at NORMAL. Valid JSON and the exactly-one-category keyword fallback are rule-validated classifications; flagged outputs and guardrail/budget cases go to human review.

## Limitations and next steps

This is a deterministic synthetic mock, not evidence of production LLM quality. The original Part B Premium flag is a post-hoc whole-window label, and any retention list built from it is illustrative rather than a leakage-free evaluation. In production, use a real tokenizer, secure transport, secret management, human auditing, and monitoring for category drift.

# Part D — Policy RAG Assistant and Refund Agent

## Retrieval evaluation

| Chunking | Expected document in top 3 | Expected fact in answer | Unanswerable fallback |
|---|---:|---:|---:|
| 40 words, overlap 10 | 12/12 | 12/12 | 2/2 |
| 40 words, overlap 0 | 12/12 | 10/12 | 2/2 |

With zero overlap, the delivery-time question “What delivery time makes a completed order eligible for a full refund?” fails the expected-fact check because the “more than 45 minutes” condition is split from the sentence fragment selected by the mock. The payment-failure question also fails the fact check because the relevant answer sentence is split across chunks.

## Agent scenarios

| Scenario | Status | Refund (INR) | Model calls | Tokens |
|---|---|---:|---:|---:|
| S1 | REFUND | 128.25 | 5 | 957 |
| S2 | REFUND | 135.00 | 5 | 957 |
| S3 | ESCALATE | 1999.00 | 5 | 944 |
| S4 | DENIED | 0.00 | 4 | 653 |
| S5 | DENIED | 0.00 | 3 | 373 |
| S6 | NOT_FOUND | 0.00 | 2 | 96 |
| S7 | FALLBACK_MAX_ITERATIONS | 0.00 | 6 | 3072 |

## Safety and resume demos

- Token limit: **FALLBACK_TOKEN_LIMIT**, 3 calls, 390 tokens.
- Timeout: **FALLBACK_TIMEOUT**, 0 calls, 0 tokens.
- Crash/resume: **REFUND**, refund INR 128.25; tools: `get_order → search_policy → calculate_refund → issue_refund`.

The full seven-scenario table, implementation explanation, TF-IDF description, and ReAct versus plan-and-execute comparison are in the project README. All model calls use the local deterministic mock; no API key or network request is used.

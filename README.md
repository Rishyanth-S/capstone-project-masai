# Instamart Customer Intelligence & Support Copilot

## Overview

This repository is one internal Copilot for Swiggy Instamart's customer-experience team. It fixes three problems in one pipeline:

1. **Trusted data.** Raw order exports are audited, cleaned with documented rules, stored in SQLite, and queried in SQL and pandas.
2. **Retention.** Customers are segmented with RFM and K-means, and a supervised model scores how likely each customer is to lapse.
3. **Support.** Tickets are privacy-redacted, classified with guardrails and retries, and routed using the customer segments. A policy RAG assistant and a tool-using refund agent complete the Copilot.

All data is synthetic and fictional. The two supplied files, `generate_data.py` and `mock_llm.py`, are used unmodified. Every model call is offline through `mock_llm.py`; no API key, account, network call or paid service is used. All monetary values are in Indian Rupees (INR).

**Repository layout**

```
generate_data.py, mock_llm.py, requirements.txt, .gitignore, README.md
data/raw/                  generated raw CSVs and policy text files
data/clean/                cleaned customers.csv, products.csv, orders.csv
data/instamart.db          SQLite database (customers, products, orders)
data/policies/             five supplied policy documents
sql/queries.sql            seven Part A queries
src/part_a.py              audit, clean, store, SQL and pandas analysis
src/validate_part_a.py     standalone Part A checkpoint validator
src/part_b.py              RFM, K-means, lapse model, retention list
src/part_c.py              ticket guardrails, redaction, triage and routing
src/part_d.py              policy RAG, refund tools, ReAct agent, resume
outputs/                   required CSV, joblib and JSONL deliverables
state/                     agent checkpoints for resume
```

**Stack:** Python 3.12, pandas, numpy, scikit-learn, joblib, sqlite3, and the standard library. No frontend is required by this brief; the deliverable is text and code only.

### Academic integrity and AI-use disclosure

This repository is built for the capstone brief. The two supplied files, `generate_data.py` and `mock_llm.py`, are used unmodified; all Part A–D code, SQL and documentation were written for this repository. The data is synthetic, no real customer data is present, and no network call, API key or paid service is used anywhere. `COMPLIANCE.md` records the full checklist and `src/compliance_check.py` verifies it automatically.

This repository does **not** claim that no AI assistance was used. Complete the disclosure template in `COMPLIANCE.md` honestly and follow your institution's policy. If your programme prohibits AI assistance, do not submit this repository as your own work; use it only as a reference.

## Reproduce

Run these commands in order from a clean clone. The commands work on Windows PowerShell and on macOS/Linux (use the activation line for your platform).

```bash
1.  git clone <your-public-repository-url> && cd <repository-folder>
2.  python -m venv venv
3.  source venv/bin/activate          # Windows: venv\Scripts\activate
4.  pip install -r requirements.txt
5.  python generate_data.py
6.  python src/part_a.py
7.  python src/validate_part_a.py
8.  python src/part_b.py
9.  python src/part_c.py
10. python src/part_d.py
11. python src/compliance_check.py
```

`python generate_data.py` must end with exactly:

```
generated: customers=1200 products=40 orders_raw=13084 tickets=164
```

`python src/validate_part_a.py` must end with exactly `ALL CHECKPOINTS PASSED`. Parts B, C and D print their checkpoints and write the required files under `outputs/` and `state/`. Total runtime is about two to three minutes on a normal laptop.

## Part A

### A2. Raw-data audit

The raw tables have these shapes and data types (pandas `read_csv`; tickets read with `keep_default_na=False`):

| Table | Shape | Column data types |
|---|---|---|
| customers | 1200 × 5 | customer_id object, city object, signup_date object, age float64, gender object |
| products | 40 × 4 | product_id object, product_name object, category object, price_inr int64 |
| orders | 13084 × 9 | order_id object, customer_id object, product_id object, quantity int64, order_ts object, status object, discount_pct int64, order_amount object, delivery_minutes float64 |
| tickets | 164 × 5 | ticket_id object, customer_id object, order_id object, text object, true_category object |

Missing values, for every column:

| Table.column | Missing | Missing % |
|---|---:|---:|
| customers.customer_id | 0 | 0.00 |
| customers.city | 0 | 0.00 |
| customers.signup_date | 0 | 0.00 |
| customers.age | 61 | 5.08 |
| customers.gender | 30 | 2.50 |
| products.product_id | 0 | 0.00 |
| products.product_name | 0 | 0.00 |
| products.category | 0 | 0.00 |
| products.price_inr | 0 | 0.00 |
| orders.order_id | 0 | 0.00 |
| orders.customer_id | 0 | 0.00 |
| orders.product_id | 0 | 0.00 |
| orders.quantity | 0 | 0.00 |
| orders.order_ts | 0 | 0.00 |
| orders.status | 0 | 0.00 |
| orders.discount_pct | 0 | 0.00 |
| orders.order_amount | 53 | 0.41 |
| orders.delivery_minutes | 2541 | 19.42 |
| tickets.ticket_id | 0 | 0.00 |
| tickets.customer_id | 0 | 0.00 |
| tickets.order_id | 0 | 0.00 |
| tickets.text | 0 | 0.00 |
| tickets.true_category | 0 | 0.00 |

Specific audit figures:

| Audit figure | Value |
|---|---:|
| Raw order rows | 13,084 |
| Exact duplicate order rows | 119 |
| Rows with status `pending` | 656 |
| Distinct `order_id` values | 12,309 |
| `order_amount` entries that cannot be converted to a number | 120 |
| Negative `delivery_minutes` | 106 |
| Blank `delivery_minutes` | 2,541 |
| Blank `age` | 61 |
| Ages outside 10–100 | 6 |
| Distinct raw city spellings | 24 |
| Distinct non-blank raw gender spellings | 10 |
| Blank `gender` | 30 |

### A3. Cleaning customers

| Rule | Values changed / result |
|---|---|
| Strip leading/trailing spaces and apply title case to `city` | 476 values changed; exactly six cities remain: Pune 227, Delhi 210, Bengaluru 197, Chennai 191, Hyderabad 191, Mumbai 184 |
| Map gender spellings to `male`, `female`, `unknown` (blank → unknown) | 950 values changed, including 30 blanks → unknown; result: female 610, male 560, unknown 30 |
| Treat ages outside 10–100 as missing, then fill missing ages with the median age of the customer's city using `groupby` + `transform` | 6 invalid ages became missing, joining 61 already blank; 67 ages filled; 0 missing ages remain; final range 18–60 |
| Convert `signup_date` to datetime | 0 values lost; stored as `YYYY-MM-DD` |

### A4. Cleaning orders (rules applied in this order)

| Rule | Rows / values changed |
|---|---|
| Convert `order_ts` to datetime | 13,084 rows parsed |
| Remove exact duplicate rows | 119 removed, leaving 12,965 |
| For each `order_id`, keep only the record with the latest `order_ts` (removes the earlier `pending` copy) | 656 older records removed, leaving 12,309 rows (one per order) |
| Convert `order_amount` to numeric, turn unparseable entries into missing, then fill from `round(price_inr × quantity × (1 − discount_pct / 100), 2)` | 112 missing amounts after deduplication (120 existed in the raw file; 8 were inside removed duplicate rows) filled; 0 missing remain |
| Set negative `delivery_minutes` to missing | 105 negative values after deduplication (106 in the raw file; 1 was in a removed row) |
| For `completed` orders, fill missing delivery time with the median delivery time of that order's product category among completed orders | 774 completed orders filled |
| For `cancelled` orders, leave delivery time missing | 1,199 cancelled orders remain missing; no delivery time is invented |

Final state: 12,309 rows, 11,110 completed and 1,199 cancelled, no `pending` rows, 0 missing `order_amount`, 0 missing `delivery_minutes` among completed orders, 1,199 missing among cancelled orders, and completed revenue of **INR 2,998,668.80**.

### A5. Store and validate

The three clean tables are written to `data/clean/` and to `data/instamart.db` as `customers`, `products` and `orders`. `order_ts` is stored as text in `YYYY-MM-DD HH:MM:SS` form. `src/part_a.py` asserts the checkpoints inline; `src/validate_part_a.py` independently re-reads the CSVs and database, asserts 14 checkpoints and prints `ALL CHECKPOINTS PASSED`.

### A6. SQL analysis

All seven queries are saved in `sql/queries.sql` and executed with `pd.read_sql_query`.

| Query | Technique | Reference result |
|---|---|---|
| Q1 | WHERE then HAVING | 30 customers |
| Q2 | Three-table join | 30 rows; revenues sum to INR 2,998,668.80; largest cell is Bengaluru / Electronics at INR 233,909.45 |
| Q3 | Scalar subquery | 11 products above the average catalogue price |
| Q4 | LEFT JOIN | 25 customers with no completed order |
| Q5 | Two CTEs | 550 customers above the average total spend; average INR 2,552.06 |
| Q6 | RANK, DENSE_RANK, ROW_NUMBER | Top customer per city: Bengaluru C0082 (INR 12,856.20), Chennai C0930 (INR 10,151.40), Delhi C0291 (INR 13,742.60), Hyderabad C0743 (INR 12,730.05), Mumbai C0693 (INR 14,766.55), Pune C0426 (INR 10,377.10) |
| Q7 | Running total with a window | 183 daily rows; last cumulative value INR 2,998,668.80 |

Exactly **8 rows** have `RANK()` different from `ROW_NUMBER()`: customers in the same city with identical total spend share a `RANK`, while `ROW_NUMBER` breaks the tie arbitrarily; `RANK` also leaves a gap after a tie, whereas `DENSE_RANK` does not.

### A7. pandas cross-check and exploratory analysis

The `pd.pivot_table(..., margins=True)` grand total is **INR 2,998,668.80**, identical to the SQL total. Per-city completed-order statistics:

| City | Sum (INR) | Mean (INR) | Count |
|---|---:|---:|---:|
| Bengaluru | 523,643.40 | 276.47 | 1,894 |
| Chennai | 463,629.15 | 272.72 | 1,700 |
| Delhi | 502,829.80 | 243.62 | 2,064 |
| Hyderabad | 528,589.90 | 284.95 | 1,855 |
| Mumbai | 479,058.50 | 297.00 | 1,613 |
| Pune | 500,918.05 | 252.48 | 1,984 |

Completed `order_amount` statistics: mean INR 269.91, median INR 135.00, skewness 4.61, Q1 INR 64.80, Q3 INR 297.00, IQR INR 232.20, lower fence INR −283.50, upper fence INR 645.30, **897 outliers above** the upper fence and **0 below** the lower fence.

**What is this data about?** It is a synthetic six-month history of grocery, dairy, snack, personal-care and small-electronics orders across six Indian cities, plus customer profiles and support tickets. **What do the distributions look like?** Completed order value is strongly right-skewed: the mean (INR 269.91) is about twice the median (INR 135.00), with a long tail of large electronics baskets. **Are there outliers or anomalies?** Yes: 897 high-value orders sit above the 1.5×IQR upper fence, and the raw exports also contained duplicates, pending copies, unparseable amounts and negative delivery times, all handled in cleaning. **What relationships exist?** The largest absolute correlation among `order_amount`, `discount_pct`, `quantity` and `delivery_minutes` is `order_amount` with `discount_pct` at about −0.095; `order_amount` with `quantity` is +0.087. A correlation of this size is very weak and **does not establish causation**.

## Part B

### B1. RFM table

As of the snapshot **2025-10-01 00:00:00**, using completed orders only: **1,175 customers**; recency median 19 days, minimum 0, maximum 180; frequency mean 9.46, maximum 33; monetary mean INR 2,552.06, maximum INR 14,766.55.

### B2. K-means

`StandardScaler` was fitted on the three RFM columns, then `KMeans(n_clusters=K, n_init=10, random_state=42)` was fitted for K = 2…8.

| K | Inertia | Silhouette |
|---:|---:|---:|
| 2 | 1820.8 | 0.465 |
| 3 | 1268.2 | 0.445 |
| 4 | 743.1 | 0.526 |
| 5 | 617.4 | 0.480 |
| 6 | 503.2 | 0.482 |
| 7 | 408.5 | 0.416 |
| 8 | 366.5 | 0.412 |

The elbow is around K = 4, and K = 4 also has the best silhouette score (0.526). **Final K = 4**, because it is both the strongest metric and the most business-interpretable set of four actionable groups. Inertia always falls as K rises because more centroids can always fit the same points more closely; K = 8 therefore has lower inertia than K = 2, but inertia alone cannot choose K because it rewards complexity and would always pick the largest K. Silhouette and business usefulness are needed to stop.

### B3. Segment profile, names and PCA

Cluster numbers are only labels with no ordering.

| Segment id | Business name | Size | Mean recency (days) | Mean frequency | Mean monetary (INR) | Recommended action |
|---:|---|---:|---:|---:|---:|---|
| 0 | Regular | 436 | 28.86 | 3.77 | 1,057.62 | Seasonal snack and dairy offers to grow basket size |
| 1 | **Premium** | 138 | 24.12 | 6.77 | 7,129.70 | Exclusive early access and concierge service |
| 2 | Frequent Families | 419 | 8.77 | 19.40 | 3,410.02 | Weekly grocery bundles and a loyalty multiplier |
| 3 | At Risk | 182 | 109.04 | 2.20 | 685.99 | Win-back message with a small time-limited coupon |

The **Premium** flag is true for exactly segment 1, the segment with the highest mean monetary value.

`PCA(n_components=2)` on the scaled RFM matrix gives explained variance ratios of **0.634** and **0.218**, total **0.852**.

`outputs/customer_segments.csv` has columns `customer_id, recency, frequency, monetary, segment_id, segment_name, is_premium` and 1,175 rows.

### B4. Lapse problem

Cutoff: **2025-09-01**. The cohort is every customer with at least one completed order before the cutoff. `lapsed = 1` when the customer has no completed order from 2025-09-01 through 2025-09-30.

| Item | Value |
|---|---:|
| Cohort customers | 1,156 |
| Lapsed customers | 443 (38.32%) |
| Training rows | 924 |
| Test rows | 232 |

Features use **only orders dated before 2025-09-01** plus the clean customer table: recency, frequency, monetary, average discount, cancellation rate, average delivery minutes, most frequent product category, city, age, gender and tenure. The September label window, and the segment labels from task 3, are **not** used as features. There are 0 missing feature values.

### B5. Preprocessing without leakage

A `ColumnTransformer` inside a scikit-learn `Pipeline` one-hot encodes `city`, `gender` and `favourite_category` with `drop="first"` and standardises the numeric features. It is fitted on the 924 training rows only and merely applied to the 232 test rows. The scaler's learned mean for `frequency` is **8.1147**, while the full-cohort mean is **8.0285**; they differ because the scaler saw only the training sample, not the full cohort, so ordinary sampling variation moves the mean slightly.

Baseline on the untouched test set (always predict "not lapsed"): accuracy **61.6%**, lapsed-class precision 0, recall 0, F1 0.

### B6. Tune and evaluate

| Model | Grid | Best parameters | Best cross-validated F1 |
|---|---|---|---:|
| Logistic regression (`max_iter=1000`) | C ∈ {0.01, 0.1, 1, 10, 100}, cv=5, scoring=f1 | C = 100 | 0.6506 |
| Random forest (`n_estimators=100`, `random_state=42`) | `max_depth` ∈ {3, 5, 8} × `min_samples_leaf` ∈ {5, 10, 20}, cv=5, 45 fits | `max_depth=8`, `min_samples_leaf=10` | 0.6811 |

Untouched test-set results for the lapsed (positive) class:

| Model | Confusion matrix `[[TN, FP], [FN, TP]]` | Precision | Recall | F1 |
|---|---|---:|---:|---:|
| Logistic regression | [[111, 32], [31, 58]] | 0.644 | 0.652 | 0.648 |
| Random forest | [[113, 30], [31, 58]] | 0.659 | 0.652 | 0.655 |

**Selected model: random forest.** It has the slightly higher test F1 (0.655 versus 0.648) and precision (0.659 versus 0.644) at the same recall, and it captures non-linear interactions among frequency, monetary value and recency. Missing a customer who is about to lapse is expensive because the retention team loses the whole future basket, while wrongly targeting a loyal customer costs only a small campaign incentive. The extra false positives are therefore acceptable, and the forest gives the best balance of the two errors. Both models clear the required test F1 of 0.55.

### B7. Class-balance experiment

Only the training set was over-sampled: all 924 original rows were kept, and lapsed rows were duplicated with replacement (`random_state=42`) until both classes had **570** rows. The best logistic regression was refitted on this balanced training set and evaluated on the same, unchanged test set.

| Logistic model | Precision (lapsed) | Recall (lapsed) | F1 (lapsed) |
|---|---:|---:|---:|
| Unbalanced | 0.644 | 0.652 | 0.648 |
| Balanced | 0.589 | 0.854 | 0.697 |

Recall moved most, rising from 0.652 to 0.854, while precision fell from 0.644 to 0.589. For the retention team this means the balanced model finds many more true lapsers, at the cost of more false alarms; that is usually the right trade when a missed customer is more expensive than a wasted coupon.

### B8. Interpret, persist and use

Top five random-forest feature importances:

| Feature | Importance |
|---|---:|
| frequency | 0.2796 |
| monetary | 0.2712 |
| recency | 0.1151 |
| average delivery minutes | 0.0613 |
| cancellation rate | 0.0523 |

Logistic-regression coefficient signs:

| Feature | Coefficient | Business reading |
|---|---:|---|
| recency | +0.0396 | A longer gap since the last order raises the lapse probability |
| frequency | −1.6982 | More completed orders strongly lower the lapse probability |
| monetary | +0.2846 | After controlling for frequency, high-value but infrequent buyers are more likely to lapse |

The selected model is saved with `joblib.dump` to `outputs/lapse_model.joblib`, reloaded with `joblib.load`, and the reloaded model reproduces the original test predictions exactly (`True`).

`outputs/retention_list.csv` ranks test-set customers in the Premium segment by predicted lapse probability:

| customer_id | lapse_probability | actually_lapsed |
|---|---:|---:|
| C0862 | 0.4910 | 0 |
| C1112 | 0.4841 | 0 |
| C0060 | 0.4798 | 0 |
| C0802 | 0.4796 | 0 |
| C1008 | 0.4778 | 0 |
| C1000 | 0.4624 | 0 |
| C0525 | 0.4597 | 1 |
| C0133 | 0.4562 | 0 |
| C0267 | 0.4525 | 0 |
| C0167 | 0.4522 | 0 |

**1 of the 10** truly lapsed. The Premium flag comes from the whole-window RFM in task 3, which includes the September window that defines the label, so it is a **post-hoc whole-window label** and this retention list is illustrative, not a leakage-free evaluation. The model's own features still use only orders before the cutoff.

**Leakage statement:** no encoder, scaler or resampler saw any test row, and no data from 2025-09-01 onwards entered the features. The segment labels from task 3 were not used as features.

## Part C

### C1. Input guardrails

| Check | Result |
|---|---:|
| Empty or whitespace-only tickets → `skipped_empty`, no model call | 4 |
| `est_tokens(text) > MAX_INPUT_TOKENS = 400` → `flagged_long`, no model call | 4 |
| Token estimate of each long ticket | 734 |

### C2. Redaction

Patterns are applied in this order: email → `[EMAIL]`, phone → `[PHONE]`, order id → `[ORDER_ID]`.

| PII type | Substitutions across all 164 tickets | Left after redaction |
|---|---:|---:|
| Email | 55 | 0 |
| Phone | 91 | 0 |
| Order id | 132 | 0 |
| **Total** | **278** | **0** |

The user message sent to the model is exactly the redacted ticket text and nothing else. The code asserts after redaction that none of the three patterns can be found again.

### C3. Two prompts

Both system prompts contain the word `CLASSIFY`, list the four allowed categories (`billing`, `delivery`, `technical`, `fraud`) and require JSON only in the form `{"category": "...", "reason": "..."}`. The zero-shot prompt contains no `EXAMPLES:` text. The few-shot prompt contains an `EXAMPLES:` line followed by four labelled examples, including a fraud example.

### C4. Resilient calls

Every call is wrapped in a retry function with at most 3 retries. The wait before retry number n is `0.05 × 2**n × random.uniform(0.9, 1.15)`. A full run over the 156 eligible tickets produces exactly **19 `TransientLLMError` failures**, all recovered. Every retry in this mock is retry number 0, so every wait lies between **0.045 and 0.0575 seconds**; one log line is printed per retry with the ticket id, retry number and wait. `mock_llm.reset()` is called immediately before each full run. No ticket ends with a permanent error.

### C5. Validate every answer

| Status | Count | Meaning |
|---|---:|---|
| `classified_json` | 135 | Valid JSON with an allowed category |
| `classified_keyword` | 12 | Not JSON, but exactly one allowed category word found with `re.findall(r"[a-z]+", text.lower())` |
| `flagged_invalid_output` | 9 | Valid JSON with a category outside the list, or no single allowed category word |
| `flagged_long` | 4 | Input guardrail |
| `skipped_empty` | 4 | Input guardrail |

The 12 keyword replies are the mock's chatty answers of the form "The customer appears to have a `<category>`-related problem." A plain whitespace split would not separate `billing-related` into words and would give 135 / 0 / 21 instead of 135 / 12 / 9.

### C6. Zero-shot versus few-shot

| Prompt | Classified tickets | Accuracy on classified | Total tokens | Tokens per classified ticket |
|---|---:|---:|---:|---:|
| Zero-shot | 147 | 85.7% (126/147) | 10,336 | 70.3 |
| Few-shot | 147 | 100.0% (147/147) | 23,596 | 160.5 |

Zero-shot confusion table (classified tickets only):

| True category | Predicted billing | Predicted delivery | Predicted fraud | Predicted technical |
|---|---:|---:|---:|---:|
| billing | 38 | 0 | 0 | 0 |
| delivery | 0 | 36 | 0 | 0 |
| fraud | 11 | 0 | 23 | 0 |
| technical | 10 | 0 | 0 | 29 |

Every zero-shot error is a fraud ticket predicted as billing (11) or a technical ticket predicted as billing (10). The few-shot prompt costs about **90 more tokens per classified ticket** (160.5 versus 70.3, roughly 2.3× the total token bill). It is worth paying when a wrong route is expensive, as it is for fraud and premium support: the extra examples remove all 21 errors here. For a high-volume, low-risk queue where billing-like words dominate, the cheaper zero-shot prompt may be adequate, but this support desk should use few-shot for fraud-sensitive routing.

### C7. Budget, routing and output

With `TOKEN_BUDGET = 6000` on the zero-shot pipeline, **65 tickets** end `budget_exhausted`; the run spends 6,036 tokens and saves `outputs/triage_budget_demo.csv`. At least one ticket is exhausted, so the budget is demonstrated.

The final `outputs/triage_results.csv` is a few-shot run with `TOKEN_BUDGET = 100000` and has **164 rows**. Routing uses `outputs/customer_segments.csv`:

| Route | Priority | Tickets |
|---|---|---:|
| `support_queue` | NORMAL | 102 |
| `security_team` | HIGH | 34 |
| `human_review` | NORMAL | 17 |
| `premium_desk` | HIGH | 11 |

Tickets with `skipped_empty`, `flagged_long`, `flagged_invalid_output` or `budget_exhausted` go to `human_review`. Otherwise fraud goes to `security_team`, premium customers go to `premium_desk`, and everyone else goes to `support_queue`. A customer with no segment row is treated as not premium. The **147 classified tickets** were rule-validated (135 JSON + 12 keyword); the **17 human-review tickets** are the 4 empty, 4 long, 9 invalid-output tickets, and any budget-exhausted tickets in the demo.

## Part D

### D1. Chunking

Documents are split into 40-word chunks with 10-word overlap. The first chunk covers words 0–39, the next starts 30 words later, and the process stops after the first chunk whose end reaches or passes the last word.

| Source | Chunks |
|---|---:|
| refund_policy.txt | 4 |
| delivery_policy.txt | 3 |
| escalation_policy.txt | 3 |
| payment_policy.txt | 3 |
| privacy_policy.txt | 2 |
| **Total** | **15** |

### D2. Vectorise and retrieve

`TfidfVectorizer()` is fitted on the 15 chunk texts, the query is transformed with the same fitted vectorizer, and chunks are ranked by `cosine_similarity`. `retrieve(query, k=3)` returns `source`, `score` and `text`. For `refund policy late delivery`, all top three chunks come from `refund_policy.txt`.

**TF and IDF.** TF measures how often a term appears in one chunk, so a chunk that repeats a useful word scores higher. IDF measures how rare a term is across all chunks, so common words such as "the" contribute little while distinctive words such as "unauthorized" contribute a lot. A limitation of TF-IDF is that it matches words, not meaning: "money returned after a failed payment" and "amount credited within 48 hours" may share few words even though they mean the same thing.

### D3. Augment and answer

`answer(question)` retrieves three chunks, then sends two messages to `mock_llm.chat`: a system message containing `POLICY ASSISTANT` that tells the model to answer only from the supplied context and to say it could not find the information otherwise, and a user message of exactly this form:

```
CONTEXT:
[source file name]
chunk text
[source file name]
chunk text
[source file name]
chunk text
QUESTION:
question
```

It returns the model's text and the sources used.

### D4. Evaluate retrieval and grounding

| Chunking | Answerable: expected doc in top 3 | Answerable: expected fact in answer | Unanswerable: fallback |
|---|---:|---:|---:|
| 40 words, overlap 10 | 12/12 | 12/12 | 2/2 |
| 40 words, overlap 0 | 12/12 | **10/12** | 2/2 |

With zero overlap there are 13 chunks instead of 15, and two answerable questions break:

- **"What delivery time makes a completed order eligible for a full refund?"** The sentence "If a completed order is delivered in more than 45 minutes, the customer receives a full refund of the order amount" is split across a chunk boundary. The retrieved half answers only "receives a full refund of the order amount", so the expected fact "more than 45 minutes" is missing.
- **"How soon is money returned when a payment fails but is deducted?"** The phrase "within 48 hours" is split away from the rest of its sentence by a chunk boundary, so the answer does not contain "48 hours".

Overlap 10 fixes both because the important sentence appears whole in at least one chunk.

### D5. Tools and tool map

| Tool | Description | Parameters |
|---|---|---|
| `get_order` | Look up one order in the SQLite database | `order_id` |
| `search_policy` | Search the policy chunks and return the top results | `query` |
| `calculate_refund` | Apply the 45-minute rule and return `refund_inr` | `order_amount`, `delivery_minutes` |
| `issue_refund` | Append an issued refund to the ledger | `order_id`, `refund_inr` |
| `escalate_to_human` | Append an escalated refund above INR 1000 to the ledger | `order_id`, `refund_inr` |

`TOOLS_SPEC` holds the name, one-sentence description and JSON-style parameter specification for each tool. `TOOL_MAP` maps each name to its Python function. `issue_refund` and `escalate_to_human` append one JSON line with a UTC timestamp and the tool name to `outputs/refund_ledger.jsonl`.

### D6. Seven agent scenarios

The ReAct loop starts with a system message containing `AGENT` and the user message. In each iteration it calls the model; if there are no tool calls, the content is the final answer; otherwise it appends the assistant tool-call message, executes the tool, appends the JSON observation as a tool message, and loops. The normal status is the word before the colon in the final answer.

| Scenario | Status | Refund (INR) | Model calls | Tokens used |
|---|---|---:|---:|---:|
| S1 | REFUND | 128.25 | 5 | 1,157 |
| S2 | REFUND | 135.00 | 5 | 1,157 |
| S3 | ESCALATE | 1,999.00 | 5 | 1,144 |
| S4 | DENIED | 0.00 | 4 | 813 |
| S5 | DENIED | 0.00 | 3 | 493 |
| S6 | NOT_FOUND | 0.00 | 2 | 176 |
| S7 | FALLBACK_MAX_ITERATIONS | 0.00 | 6 | 3,312 |

All seven expected outcomes and refund amounts match. S1 takes exactly 5 model calls. S7 finishes all six model calls under the default `token_limit=6000` and then falls back because it never produces a final answer. `outputs/agent_runs.jsonl` has one line per scenario with `scenario_id`, `status`, `final_answer`, `llm_calls`, `tokens_used` and every step's tool name, arguments and observation.

### D7. Stop safely

| Limit / demonstration | Status | Model calls | Tools executed |
|---|---|---:|---|
| S1 with `token_limit=200` | `FALLBACK_TOKEN_LIMIT` | 2 | `get_order` |
| S1 with `time_limit_seconds=0` | `FALLBACK_TIMEOUT` | 0 | none |
| S7 with default limits | `FALLBACK_MAX_ITERATIONS` | 6 | six `search_policy` calls |

The token limit is checked after each model call and before executing the requested tool. The time limit is checked at the top of each iteration with `time.monotonic()`; a zero limit stops before the first call on every platform. Every fallback returns the steps completed so far and a short human-handoff summary in `final_answer`.

### D8. Crash and resume

S1 was run with a simulated crash immediately after the second tool execution, once its checkpoint had been saved. `resume_agent` loaded the checkpoint and continued. Tools executed across the crash and the resume, in order:

1. `get_order`
2. `search_policy`
3. `calculate_refund`
4. `issue_refund`

Exactly four tools executed, none repeated, and the final status is **REFUND** with **INR 128.25**. The checkpoint is `state/S1_crash_demo.json`.

### D9. ReAct versus plan-and-execute

The ReAct agent calls the model once per decision: S1 uses 5 calls, S5 uses 3 and S6 uses 2, because each observation changes the next action. A plan-and-execute design would ask for the whole step list once, so a clean S1-like case could use 1 planning call plus 1 final call (about 2 calls and far fewer tokens). ReAct pays more tokens but handles surprises naturally: a NOT_FOUND observation stops after one lookup (S6, 2 calls), and a cancelled order stops after the policy search (S5, 3 calls), without replanning. Plan-and-execute must replan when the first observation contradicts the plan, and a bad initial plan can waste several calls before the error is seen. For this refund workflow I would choose ReAct: correctness and safe stopping matter more than the few hundred extra tokens, the loop is short (at most 6 steps), and unexpected order states are common enough that adaptive tool choice is valuable.

## Limitations and Next Steps

- **Synthetic data.** All customers, orders, tickets, policies and prices are fictional. The findings describe the generator, not real Instamart behaviour.
- **Premium is a post-hoc whole-window label.** The Premium flag comes from RFM over the full window, including September, which is also the lapse label window. The retention list is therefore illustrative and not a leakage-free evaluation, even though the model's own features use only pre-cutoff orders.
- **One cutoff, one model family.** The lapse model is trained on a single 2025-09-01 cutoff. A real system should backtest several cutoffs, calibrate probabilities and monitor drift.
- **Segments are descriptive.** K-means on three RFM columns is simple and reproducible; segment names and actions are manual business interpretations, and cluster numbers have no ordering.
- **Mock model behaviour.** `mock_llm.py` is keyword-based and deterministic. The 100% few-shot accuracy is a property of this mock, not evidence that a hosted model would be perfect. The TF-IDF retriever also misses paraphrases, as the zero-overlap experiment shows.
- **Privacy regexes are narrow.** The three supplied patterns cover the ticket data, but a production system needs broader PII detection, redaction of names and addresses, and audit logging.
- **Refund agent scope.** The agent implements the brief's simple rules and treats every request as inside the 7-day window. It does not handle partial refunds, idempotency across repeated runs, payment-provider failures or real approval workflows. The ledger is append-only, so repeated demonstrations add lines.
- **Next steps.** Add calibrated lapse probabilities and uplift modelling, backtest the retention campaign, replace TF-IDF with a semantic retriever, expand PII detection, add a human approval queue and monitoring, and optionally put a small internal web UI (for example React) in front of the existing Python services. The repository stays on one `main` branch as required.

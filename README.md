# CityMart Retail Shop Customer Intelligence & Support Co-Pilot

An end-to-end data and AI project for a retail shop chain: clean the sales data, segment shoppers, predict who will stop visiting, triage shop support tickets automatically, and answer questions from shop policy with a refund agent that hands over to a human store manager when needed.

> CityMart is a **fictional** retail shop chain with stores in Chennai, Mumbai, Bengaluru, Delhi, Hyderabad and Pune selling Grocery, Household, Stationery, Apparel and Electronics. All data is synthetic, all money is in INR, and every model call goes through the offline `mock_llm.py` (no key, no network). Seeds are 42 everywhere.

## Use Case
A retail shop earns money from repeat shoppers, but its billing data is messy, it does not know which shoppers are valuable or drifting away, support tickets pile up at the counter and in the app, and staff answer policy questions from memory. This project builds four connected pieces: (A) a trusted sales database, (B) shopper segments and a lapse model so loyalty coupons are spent wisely, (C) automatic ticket triage, and (D) a policy-grounded co-pilot that can issue small refunds (for example when a pre-booked order keeps a customer waiting at the pickup counter) and escalates anything above INR 1000 to a human store manager.

## Learning Outcomes
- **Data cleaning and SQL:** auditing messy data, rule-based cleaning (duplicates, mixed types, outliers, inconsistent spellings), validation checkpoints, SQLite, joins, CTEs, window functions (`RANK` vs `ROW_NUMBER`).
- **Customer analytics:** RFM features, K-means and silhouette score, PCA, supervised lapse prediction with a leakage-aware split, class imbalance, persisting a model.
- **LLM engineering:** prompt design with few-shot examples, JSON parsing with keyword fallback, PII redaction, retries with back-off, token budgets.
- **RAG and agents:** chunking with overlap, TF-IDF retrieval, grounded answers and evaluation, a ReAct tool-using agent with guard-rails, human escalation, crash-safe checkpoint and resume.
- **Engineering practice:** reproducible seeds, a one-command pipeline runner, a notebook for exploration and scripts for automation, clear documentation.

## Quick Start
```bash
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
python run_project.py               # runs Parts A to D in order (about 20 seconds)
```
Options: `python run_project.py --only B C`, `--from B`, `--skip-generate`.
Then open `Retail_Shop_Customer_Intelligence_Copilot.ipynb` to explore the results of every part.

## Code Structure
| File | Purpose |
|---|---|
| `run_project.py` | Runs every stage in order, stops at the first failure, checks the expected output files exist |
| `Retail_Shop_Customer_Intelligence_Copilot.ipynb` | Notebook to explore and verify Parts A to D (tables, SQL, plots, retrieval demo) |
| `generate_data.py` | Creates the synthetic shop data: customers, products, sales, tickets, policy documents, evaluation questions and refund scenarios |
| `mock_llm.py` | Offline simulated LLM (ticket classifier, policy answerer, refund agent) |
| `src/part_a_clean.py` | Audit and cleaning rules for customers and sales, writes `data/clean/` and `data/citymart.db` |
| `src/part_a_validate.py` | 13 validation checkpoints on the clean tables |
| `src/part_a_sql.py` and `sql/queries.sql` | The 7 SQL queries plus the pandas cross-check |
| `src/part_b.py` | RFM, K-means, PCA, lapse models, class-balance experiment, retention list |
| `src/part_c.py` | Ticket redaction, classification, retry and budget handling, routing |
| `src/part_d_rag.py` | Shop-policy chunking, TF-IDF retrieval, grounded answers, evaluation |
| `src/part_d_agent.py` | ReAct refund agent: tools, limits, checkpoint and resume |
| `requirements.txt` | Python dependencies |

Folders: `data/raw` (generated), `data/clean` and `data/citymart.db` (Part A output), `data/policies` (shop policy knowledge base: refund, store and pickup, payment, privacy, escalation), `outputs/` (results of B, C, D), `state/` (agent checkpoints).

## Data Model
| Table / file | Meaning | Key columns |
|---|---|---|
| `customers` | Loyalty-card shoppers | `customer_id`, `city`, `signup_date`, `age`, `gender` |
| `products` | Items on the shop shelves | `product_id`, `product_name`, `category`, `price_inr` |
| `sales` | One row per bill (counter purchase or pre-booked pickup order) | `sale_id` (`SALE-100001`), `customer_id`, `product_id`, `quantity`, `sale_ts`, `status` (completed / cancelled), `discount_pct`, `sale_amount`, `counter_wait_minutes` |
| `tickets` | Shop support tickets | `ticket_id`, `customer_id`, `sale_id`, `text`, `true_category` (billing / store_service / technical / fraud) |

## Part Summaries
- **Part A, Data foundation:** 13,084 raw sales rows become 12,309 clean sales (119 exact duplicates and 656 older records removed, 112 unparseable amounts rebuilt from price, quantity and discount). Customers get 6 clean cities, 3 gender values and filled ages. 13 validation checks pass and the SQL total agrees with pandas (completed sales INR 5,482,397.60).
- **Part B, Segments and lapse model:** 1,175 shoppers are scored on RFM and grouped into 4 segments (Premium Big Spenders, Loyal Frequent Buyers, Occasional Low-Value, Lapsing / At-Risk). A logistic-regression model predicts lapse and a Premium retention list of 10 shoppers is produced.
- **Part C, Ticket triage:** 164 tickets are redacted, classified (billing, store_service, technical, fraud) and routed to the support queue, premium desk, security team or human review, with a token-budget demo.
- **Part D, Policy RAG and refund agent:** 5 shop-policy files are chunked and retrieved with TF-IDF; the agent looks up the sale, reads policy, calculates the refund, pays small refunds, escalates anything above INR 1000 and survives a simulated crash without double refunding.

## Reproduce
Run from the repository root, in order (or just `python run_project.py`):

1. `python -m venv venv` and `source venv/bin/activate`
2. `pip install -r requirements.txt`
3. `python generate_data.py` (last line: `generated: customers=1200 products=40 sales_raw=13084 tickets=164`)
4. `python src/part_a_clean.py`
5. `python src/part_a_validate.py` (prints `ALL CHECKPOINTS PASSED`)
6. `python src/part_a_sql.py`
7. `python src/part_b.py`
8. `python -m src.part_c`
9. `python -m src.part_d_rag`
10. `python -m src.part_d_agent`

Part C needs `outputs/customer_segments.csv` from step 7; Part D needs `data/citymart.db` from step 4. Steps 8 to 10 use `-m` so that `mock_llm.py` at the root is importable.

## Part A: Results
**Raw audit:** 13,084 sales rows, 119 exact duplicates, 656 `pending` rows (older versions of a bill), 12,309 distinct `sale_id`, 120 non-numeric `sale_amount` (`error` or `N/A`), 106 negative and 2,541 blank `counter_wait_minutes`; customers have 61 blank ages, 6 impossible ages, 24 spellings of 6 cities and 10 gender spellings.

**Cleaning rules:** cities are stripped and title-cased (476 values changed); gender is mapped to male / female / unknown (950 changed); 6 impossible ages are set to missing and 67 missing ages are filled with the city median. For sales: timestamps converted, 119 duplicates removed, the latest record kept per `sale_id` (656 older dropped), 112 bad amounts filled with `price x quantity x (1 - discount/100)`, 105 negative waits set to missing and 774 missing waits of completed sales filled with the product-category median (cancelled sales stay missing).

**Result:** 12,309 sales (11,110 completed, 1,199 cancelled), no pending, no missing amounts, completed sales total **INR 5,482,397.60**. `src/part_a_validate.py` asserts 13 checkpoints.

**SQL (`sql/queries.sql`):**
| Query | Result |
|---|---|
| Q1 WHERE + HAVING | 83 shoppers with more than INR 8,000 in big bills |
| Q2 three-table join | 30 city x category rows, total INR 5,482,397.60; largest Delhi / Household INR 326,616.45 |
| Q3 scalar subquery | 14 products above the average price |
| Q4 LEFT JOIN | 25 customers without a completed sale |
| Q5 CTEs | 573 customers above the average spend of INR 4,665.87 |
| Q6 ranking windows | top shopper per city: Bengaluru C0082 15,314.40; Chennai C0930 13,928.40; Delhi C0291 17,118.60; Hyderabad C0743 13,185.25; Mumbai C0659 16,415.50; Pune C0426 16,594.95; 7 rows where `RANK()` differs from `ROW_NUMBER()` |
| Q7 running total | 183 daily rows, last cumulative INR 5,482,397.60 |

Tied shoppers share the same `RANK()` and the next rank is skipped, while `ROW_NUMBER()` always gives distinct numbers, so the two columns disagree after a tie.

## Part B: Results
- **RFM** (snapshot 2025-10-01): 1,175 shoppers with a completed sale; recency median 19 days, frequency mean 9.46, monetary mean INR 4,665.87.
- **K-means:** silhouette is 0.475 for K=2 and 0.461 for K=4; K=4 is used because four business segments are more useful than two. PCA keeps 89.5% of the variance in two components.
- **Segments (K=4):** Lapsing / At-Risk 181 (recency 109 days), Loyal Frequent Buyers 412 (about 19.5 visits), Occasional Low-Value 418, Premium Big Spenders 164 (highest spend per shopper, INR 8,689).
- **Lapse problem:** 1,156 shoppers, of whom 443 (38.3%) bought in the past but not in the final month. Train/test split is 924 / 232 (stratified).
- **Models:** logistic regression F1 0.649 and random forest F1 0.619 on the test split; the majority-class baseline has F1 0. Over-sampling the minority class raises recall from 0.697 to 0.831 (F1 0.688) at the cost of precision. Frequency is the strongest predictor.
- **Output:** `outputs/customer_segments.csv`, `outputs/lapse_model.joblib` (reloaded predictions are identical) and `outputs/retention_list.csv` (10 Premium shoppers with the highest lapse probability, 2 of whom truly lapsed).

## Part C: Results
- Guard-rails: 4 empty tickets and 4 very long tickets are never sent to the model; email, phone and `SALE-` ids are redacted (278 substitutions, 0 left) and a privacy assertion proves no personal data reaches any of the 312 prompts.
- Retries: transient model errors are retried with exponential back-off (19 retries in the zero-shot run).
- **Zero-shot accuracy 85.2%** (121/142) versus **few-shot 100%** (142/142); few-shot costs about 2.6x more tokens (23,465 vs 8,957). Zero-shot confuses technical and fraud tickets with billing because they mention payments or charges.
- Token-budget demo (6,000 tokens): 51 tickets are left unprocessed with status `budget_exhausted`.
- **Routing (164 tickets):** support queue 94, security team 34 (all fraud, HIGH priority), human review 22, premium desk 14.

## Part D: Results
- **Chunks and retrieval:** 5 policy documents are cut into 40-word chunks with a 10-word overlap (17 chunks) and indexed with TF-IDF.
- **Evaluation (14 questions):** the right document is in the top 3 for 12/12 answerable questions, the expected fact is in the answer for 12/12, and both unanswerable questions correctly fall back to "I could not find that information". Without overlap the fact appears in only 10/12 answers, so overlap matters.
- **Agent tools:** `get_sale`, `search_policy`, `calculate_refund`, `issue_refund`, `escalate_to_human`.
- **Seven scenarios all match expectations:** two automatic refunds (INR 282.15 and 162.00), one escalation to a human store manager (INR 1,852.50), two denials (counter wait within 45 minutes; cancelled sale), one not-found sale, and one looping request that stops with `FALLBACK_MAX_ITERATIONS`.
- **Safety limits:** token limit (200) and time limit (0 s) demos hand over to a human; a simulated crash after the second tool resumes from the checkpoint and finishes the refund without repeating tools.

## ReAct versus plan-and-execute
ReAct decides the next tool after seeing each result, which suits a refund where the next step depends on the sale status and the wait time, and it is easy to stop at any step. Plan-and-execute writes the whole plan first, which saves model calls when the steps are fixed but handles surprises (a missing sale, a cancelled sale) less gracefully.

## Limitations and Next Steps
- All data is synthetic, so the models and thresholds say nothing about real shoppers.
- The Premium flag is a post-hoc whole-window label, so the retention list is illustrative and not a leakage-free evaluation (only 2 of its 10 shoppers truly lapsed).
- The lapse model is evaluated on one 232-row test split with one seed; repeated splits, confidence intervals and a temporal split would be more reliable. The best logistic-regression C=100 is at the edge of the grid.
- Over-sampling trades precision for recall; the probability threshold should be tuned to the cost of a loyalty coupon versus a lost shopper.
- The mock classifier is keyword based, so its 100% few-shot accuracy reflects the mock, not a real model; 22 tickets always go to humans.
- TF-IDF retrieval cannot match synonyms; embeddings or hybrid search would help. The demo refund ledger is a plain file without idempotency keys.
- Next steps: a scheduled pipeline, a real hosted model behind the same interface, and drift monitoring for the lapse model.

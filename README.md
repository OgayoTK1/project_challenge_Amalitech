# Last Mile Logistics Auditor
### Delivery Performance Audit - Veridi Logistics

> **Built by Andrew Ater Ogayo · July 2026**  
> *A production-quality delivery audit pipeline connecting logistics data to customer sentiment across 99,441 real e-commerce orders.*

---

## A. Executive Summary

Analysis of **96,470 delivered orders** reveals that **8.1% arrived after the promised date** (4.4% super-late, meaning more than 5 days past the promise) - but the more surprising finding is that **90.4% of orders arrive early, a median of 11 days early**. This means the delivery-promise algorithm is miscalibrated in both directions: systematically padding estimates rather than setting accurate expectations.

Late deliveries are **regionally concentrated, not nationwide**. A two-proportion z-test (z = 22.1, p ≈ 0) confirms remote states run 13.6% late versus 7.4% in the core - but the failure is driven specifically by the **Northeast corridor (Alagoas 23.9%, Maranhão 19.7%)** and **Rio de Janeiro (13.5% on 12,350 orders, producing 1,664 late deliveries)**. The far-North states paradoxically show the *lowest* late rates because their promises are padded by ~19 days - a different calibration failure that depresses conversion rates.

The sentiment link is a **cliff, not a slope**: on-time orders average **4.30 ★** while super-late orders collapse to **1.78 ★** (Pearson r = −0.267, p ≈ 0). Crossing the 5-day threshold costs 1.68 additional stars - the operational intervention point is clear. A composite **Delivery Risk Score** ranks all 27 states into actionable tiers, with Alagoas and Maranhão classified as Critical Risk and nine further states as High Risk.

---

## B. Project Links

| Deliverable | Link |
|---|---|
| **Notebook** (Google Colab) | `https://colab.research.google.com/drive/1ZZCauuifJcVYJX0NNgyQIggKzCsN4Tgq?usp=sharing` |
| **Live Dashboard** (Streamlit) | `https://ogayotk1-project-challenge-amalitech-app-suwpkg.streamlit.app/` |
| **Notebook** (jupyter html link) | 'https://nbviewer.org/github/OgayoTK1/project_challenge_Amalitech/blob/main/notebooks/last_mile_logistics_auditor%20%283%29.html' |
| **Presentation** (PDF/Slides) | ` https://canva.link/7nu78tae1flsesg ` |
| **Video Walkthrough** (optional) | `[]` |

---

## C. Technical Explanation

### Data Cleaning

Six raw Olist CSVs (443,778 total rows across tables) were loaded and profiled before any transformation:

**Duplicate detection:** Exact duplicate rows checked per table before any join — zero duplicates found, confirmed rather than assumed.

**Date parsing:** All five timestamp columns converted from strings to `datetime64[ns]` using `errors="coerce"`, so malformed values surface as `NaT` in validation rather than crashing date arithmetic downstream.

**Review deduplication (critical):** The reviews table contains 551 orders with multiple review rows - a 1-to-many trap that would silently duplicate orders and inflate every KPI if joined directly. The pipeline keeps the most recent review per order, then enforces the assumption with `merge(validate="one_to_one")` plus row-count assertions. The pipeline fails loudly if data ever violates this assumption.

**Non-delivered orders:** 1,234 canceled/unavailable orders and 8 delivered orders missing timestamps are flagged explicitly and excluded from delivery KPIs - nothing is silently dropped. Every excluded row is counted and reported.

**Two-grain architecture:** Products cannot join directly to orders (the link runs through `order_items` and is 1-to-many). An order-grain master table handles all delivery and sentiment KPIs; a separate item-grain table handles category analysis. Mixing the two grains would duplicate orders and corrupt every aggregate.

### Choice - Delivery Risk Score

The **Delivery Risk Score** is a composite metric that ranks every state into a single, actionable priority list:

```
Risk Score = 0.5 × Late Rate + 0.3 × Avg Delay + 0.2 × Low-Review %
```

Each component is **min-max normalised to [0, 1]** before weighting - a percentage and a day-count cannot be meaningfully summed on raw scales. The normalised components are then weighted by business priority (promise breach > magnitude of failure > reputational damage already done) and rescaled to 0–100. States are tiered at fixed cut-offs: **Low (<25) · Medium (25–50) · High (50–75) · Critical (>75)**.

**Why this matters:** Late rate alone would rank Alagoas (397 orders) equally urgent to Rio de Janeiro (12,350 orders) - the composite score, combined with volume context, lets executives sequence carrier renegotiations and fulfillment node investments by total expected impact rather than by rate alone. Recomputed monthly, it becomes the KPI that tracks whether interventions are working.

**Result:** Alagoas (100.0) and Maranhão (85.9) are Critical Risk. Nine further states — SE, CE, PI, BA, RJ, ES, PA, MS, TO — are High Risk.

---

## Key Findings

| # | Finding | Evidence |
|---|---|---|
| 1 | **The promise is padded, not calibrated.** 90.4% of orders arrive early (median 11 days), yet 8.1% still miss the date. Both tails are business problems. | Delay distribution; median delay_days = −11 |
| 2 | **Failure is regionally concentrated.** Northeast + RJ drive failures; far North is over-padded. Not a nationwide problem. | Two-proportion z-test: z = 22.1, p ≈ 0 |
| 3 | **The sentiment cliff is at 5 days.** 1–5 days late costs 0.70 stars. Past 5 days costs a further 1.68 stars - Super-Late averages 1.78★. | Review by delivery status |
| 4 | **Audio and fashion categories structurally underperform.** audio (12.7%), fashion_underwear_beach (12.6%), christmas_supplies (12.0%) run ~1.5× the national rate. | Category late-% ranking |
| 5 | **AL and MA are Critical Risk; RJ is the volume priority.** 1,664 late deliveries from RJ alone exceed the combined late count of most Northeast states. | Delivery Risk Score |

---

## Business Recommendations

| Priority | Action | Evidence |
|---|---|---|
| **CRITICAL** | Recalibrate the promise algorithm. Tighten far-North padding (~19d excess); add buffer in Northeast where we chronically miss. | Delay distribution; state avg_delay_days |
| **CRITICAL** | Launch Northeast + RJ intervention programme. Sequence: (1) RJ carrier SLA review for volume impact, (2) AL/MA carrier renegotiation or Recife/Fortaleza distribution point. | Risk Score; z-test |
| **HIGH** | Defend the 5-day line with proactive comms. Automated ETA revision + goodwill credit before the promise is missed - triggered when a delivery is at risk of crossing the Super-Late threshold. | Review by status; dose-response scatter |
| **MEDIUM** | Category-specific carrier SLAs for audio, fashion_underwear_beach, christmas_supplies. | Category ranking |
| **MEDIUM** | Adopt Delivery Risk Score as a monthly operations KPI. Target: AL and MA exit Critical tier within 6 months. | Choice |

---

## Architecture

```
┌────────────────────── EXTRACT ──────────────────────┐
│  6 raw Olist CSVs  →  load_datasets()               │
└──────────────────────────┬──────────────────────────┘
┌────────────────────── VALIDATE ─────────────────────┐
│  audit_dataframe(): missing %, dtypes, duplicates   │
└──────────────────────────┬──────────────────────────┘
┌────────────────────── TRANSFORM ────────────────────┐
│  clean_table(): dedupe + date parsing               │
│  deduplicate_reviews(): 1 review per order          │
│  build_master_dataset(): validated joins            │
│      ORDER grain  +  build_item_dataset() ITEM grain│
│  add_delivery_features(): Days_Difference, status   │
└──────────────────────────┬──────────────────────────┘
┌────────────────────── ANALYZE ──────────────────────┐
│  State metrics · z-test · sentiment correlation     │
│  Category metrics · Delivery Risk Score · KPIs      │
└──────────────────────────┬──────────────────────────┘
┌────────────────────── LOAD ─────────────────────────┐
│  data/processed/*.csv  →  Streamlit dashboard       │
└─────────────────────────────────────────────────────┘
```

## ETL Join Design

| Join | Key | Cardinality | Type | Safeguard |
|---|---|---|---|---|
| orders ← customers | `customer_id` | 1:1 per order | LEFT | `validate="one_to_one"` + row-count assert |
| orders ← reviews (deduped) | `order_id` | 1:1 after dedupe | LEFT | dedupe to latest + `validate="one_to_one"` |
| order_items ← products | `product_id` | many:1 | LEFT | `validate="many_to_one"` |
| order_items ← translation | `product_category_name` | many:1 | LEFT | fallback to original Portuguese name |

All joins use pandas `validate=` parameter — merges **fail loudly** on assumption violations rather than silently producing duplicate rows.

---

## Dataset

| Table | Rows | Columns | Description |
|---|---|---|---|
| `olist_orders_dataset.csv` | 99,441 | 8 | Central orders table — status, timestamps |
| `olist_order_reviews_dataset.csv` | 99,224 | 7 | Customer review scores and text |
| `olist_customers_dataset.csv` | 99,441 | 5 | Customer location (state, city) |
| `olist_products_dataset.csv` | 32,951 | 9 | Product metadata and categories |
| `olist_order_items_dataset.csv` | 112,650 | 7 | Order-to-product link table |
| `product_category_name_translation.csv` | 71 | 2 | Portuguese → English category names |

**Source:** [Olist Brazilian E-Commerce Dataset — Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

> Raw CSVs are **not committed to this repository** (gitignored). Download from Kaggle and place in `data/raw/`.

---

## Executive KPIs (from pipeline output)

| KPI | Value |
|---|---|
| Total Orders | 99,441 |
| Delivered Orders | 96,470 |
| Canceled / Unavailable | 1,234 |
| Late Orders % | 8.11% |
| Super Late % (>5 days) | 4.37% |
| Average Delay — all delivered | −10.88 days |
| Median Delay — all delivered | −11.0 days |
| Average Delay when Late | +9.87 days |
| Average Review Score | 4.16 ★ |
| Worst State (late %) | AL — 23.9% |
| Best State (late %) | RO — 2.9% |
| Worst Product Category | audio — 12.7% |
| Best Product Category | books_imported — 3.5% |

---

## Repository Structure

```
last-mile-logistics-auditor/
│
├── app.py                          ← Streamlit dashboard (run: streamlit run app.py)
├── requirements.txt                ← Python dependencies
├── README.md                       ← This file
├── .gitignore                      ← Excludes raw CSVs and large files
│
├── notebooks/
│   ├── last_mile_logistics_auditor.ipynb    ← Main analysis notebook
│   └── last_mile_logistics_auditor.html     ← HTML export (charts visible without Colab)
│
└── data/
    ├── raw/                        ← gitignored — download from Kaggle
    └── processed/                  ← pipeline-generated CSVs for dashboard
        ├── master_delivery_dataset.csv
        ├── state_performance.csv
        ├── state_risk_ranking.csv
        ├── category_performance.csv
        └── executive_kpis.csv
```

---

## How to Run

### Notebook (Google Colab — recommended)
1. Open `notebooks/last_mile_logistics_auditor.ipynb` in Google Colab
2. Upload the 6 Olist CSVs to your Drive under `data/`
3. **Runtime → Run All**
4. Processed CSVs are written to `data/processed/` automatically

### Dashboard (local)
```bash
git clone https://github.com/YOUR_USERNAME/last-mile-logistics-auditor
cd last-mile-logistics-auditor
pip install -r requirements.txt
streamlit run app.py
```

The dashboard runs fully on hardcoded aggregates — no CSVs required for the core charts.

### Deploy dashboard (Streamlit Cloud)
1. Push repo to GitHub
2. Go to [share.streamlit.io](https://share.streamlit.io) → New app
3. Select repo · Branch: `main` · Main file: `app.py`
4. Deploy → public URL in ~60 seconds

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Python 3.11 | Core language |
| pandas | Data manipulation, joins, aggregations |
| numpy | Numerical operations, normalisation |
| scipy | Two-proportion z-test, Pearson/Spearman correlation |
| plotly | Interactive charts |
| streamlit | Live public dashboard |
| Google Colab | Cloud notebook environment |
| GitHub | Version control and portfolio hosting |

---

## Limitations & Future Improvements

| Limitation | Implication |
|---|---|
| Correlation ≠ causation | Review scores also reflect product quality, pricing and seller behaviour. A regression model with category and seller fixed effects is the rigorous next step. |
| Small-sample states (RR 41, AP 67, AC 80) | Rates are directional, not precise. Excluded from best/worst rankings. |
| No carrier or seller IDs in scope | Cannot attribute late deliveries to specific carriers without additional data. |
| Static historical data (2016–2018) | Production use requires scheduling (Airflow/dbt) with live data feeds. |

**Future work:**
- NLP on `review_comment_message` to separate "late delivery" complaints from "broken product" complaints
- Carrier-level attribution once carrier join keys are available
- Freight-cost vs speed trade-off analysis
- Productionise as an Airflow DAG with dbt models and Great Expectations data quality tests

---



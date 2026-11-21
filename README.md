# Decoding Customer Value: A SQL-Driven Retention Strategy

**Consulting & Analytics Club, IIT Guwahati · Summer Projects '26**

A customer-intelligence project for a D2C fashion brand (3,900 customers, US-only, no stores). Using only transactional and behavioural data, it answers one question:

> *Is the business building a loyal customer base, or is it reliant on continuous promotional activity, and what should it do about it?*

**Short answer:** it is reliant on promotions, but the problem is correctable. The project identifies exactly which customers to stop discounting first, in what order, and how to measure whether it worked.

---

## Headline findings

| Finding | Evidence |
|---|---|
| Promotions are applied to everyone, not strategically | Promo rate is flat across value tiers: Premium 44.1%, High 41.5%, Mid 44.5%, Low 41.9% (overall 43.0%) |
| 43.1% of portfolio value sits with promo users | Share of total CLV proxy from customers with `promo_dependency_score = 1` |
| A small organic core exists | 393 Brand Advocates (10.1% of base) average a CLV proxy of $1,111 vs. $716 overall, with 0% promo use |
| Purchase frequency drives value | Weekly buyers average $1,257 CLV proxy vs. $181 for Annual buyers (6.9×); spend per order barely varies |
| Subscription is not loyalty here | 100% of subscribers use a promo code, so the subscription programme and promo programme overlap completely |

## Recommendations (full detail in the Retention Playbook)

| Phase | Segment | Size | Action |
|---|---|---|---|
| 1. Sunset immediately | Premium Brand Advocates | 205 | Remove from all promo sequences; track 30-day repeat rate |
| 2. 90-day pilot | Premium and High promo users (Promo Hunters + Subscription Members) | 835 | 50/50 test vs. holdout; extend only if Day-30 repeat rate holds within 10% |
| 3. Re-evaluate | Low-tier promo users | 409 | CAC audit by Day 60; exit promos if CLV < 1.5× CAC |
| No action | Mid-tier promo users (monitor) and Passive Buyers (no promo exposure) | 433 + 1,830 | Monitor quarterly |

---

## Repository contents

### Final deliverables (submit these)

| Deliverable (per problem statement) | File |
|---|---|
| **Python**: cleaned data + engineered features | `day1_feature_engineering.py`, `customers_enriched.csv` |
| **SQL**: segmentation queries for all 5 key questions | `day2_sql_segmentation_final.py` (+ `q*.csv` and `master_segment_summary.csv` outputs) |
| **Power BI**: four-panel founder dashboard | `Customer_Intelligence_Dashboard.pbix` *(built by you in Power BI Desktop, not included in this folder)*, fed by the `pbi_*.csv` files; steps in `powerbi_build_guide.docx` |
| **Playbook**: promo sunset plan + ideal customer profile | `retention_playbook_v2.docx` |
| **Summary**: executive summary (max 1 page) | `executive_summary_v2.docx` |

### Supporting scripts

| File | Purpose |
|---|---|
| `day3_powerbi_prep.py` | Builds the pre-aggregated tables the dashboard panels read |

### Superseded drafts (safe to delete before submitting)

`day2_sql_segmentation.py`, `day2_sql_segmentation_v2.py`, `executive_summary.docx`, `executive_summary_final.docx`, `retention_playbook.docx`, `retention_playbook_final.docx`

---

## How to reproduce

**Requirements:** Python 3.9+ and `pandas`. SQLite is part of Python's standard library, so there is nothing else to install.

**Run order matters:**

```bash
python day1_feature_engineering.py     # cleans Dataset.csv -> customers_enriched.csv
python day2_sql_segmentation_final.py  # re-derives customer_segment, runs all SQL, exports q*.csv
python day3_powerbi_prep.py            # builds pbi_*.csv for Power BI
```

1. The scripts use hard-coded paths (`/mnt/user-data/...`). Edit the `RAW_PATH` / `INPUT_PATH` / `OUT` constants at the top of each file to match your machine.
2. Run Day 2 *after* Day 1. Day 2 rebuilds `customer_segment` and overwrites `customers_enriched.csv`. The `customer_segment` written by `day1_feature_engineering.py` is **superseded** (see Known limitations).
3. Run Day 3 last. It reads the Day-2 version of `customers_enriched.csv`.

---

## Data and method

**Source:** `Dataset.csv`, 3,900 rows × 18 columns (age, gender, item, category, purchase amount, location, season, review rating, subscription status, shipping type, discount applied, promo code used, previous purchases, payment method, purchase frequency). It has **no timestamps, no loyalty score and no churn label**, so every concept below is constructed from available variables.

**Cleaning:** 37 missing `review_rating` values imputed with the median. No other nulls. `discount_applied` and `promo_code_used` are identical in every row, so they are treated as a single signal.

### Engineered metrics

| Metric | Definition | Question it answers |
|---|---|---|
| `frequency_score` | Weekly=7, Bi-Weekly=6, Fortnightly=5, Monthly=4, Every 3 Months=3, Quarterly=2, Annually=1 | How often does this customer buy? |
| `promo_dependency_score` | 1 if a discount/promo code was used, else 0 | Who needs a discount to buy? |
| `satisfaction_flag` | 1 if `review_rating` ≥ 4.0 | Who is genuinely happy? |
| `clv_proxy` | `purchase_amount × frequency_score × ln(1 + previous_purchases)` | Estimated lifetime value without timestamps |
| `value_tier` | Quartiles of `clv_proxy`: Low / Mid / High / Premium | Where does a customer sit in the value distribution? |
| `rfm_segment` | RFM with `frequency_score` as the recency proxy (no dates available) | Classic segmentation cross-check |

### Two competing loyalty definitions (as required)

| | Definition | Customers | Mean CLV proxy | Correlation with CLV proxy |
|---|---|---|---|---|
| **Def 1**: commitment | `previous_purchases ≥ 25` AND subscribed | 569 | $874 | 0.12 |
| **Def 2**: organic pull *(chosen)* | Weekly / Bi-Weekly / Fortnightly AND rating ≥ 4.0 AND no promo | 393 | $1,111 | 0.25 |

Def 2 is used because it correlates about twice as strongly with value and excludes promo-driven behaviour, which is the behaviour the brand is trying to move away from.

### Customer segments (every label maps to stated variables)

| Segment | Rule (evaluated in this order) | Customers |
|---|---|---|
| Brand Advocate | `loyalty_def2 = 1` | 393 |
| Subscription Member | `subscription_status = 'Yes'` (not already an Advocate) | 1,053 |
| Promo Hunter | no subscription, `promo_dependency_score = 1` | 624 |
| Passive Buyer | no subscription, no promo | 1,830 |

### SQL layer

All queries run against an in-memory SQLite table built from `customers_enriched.csv`. The syntax is standard (window functions, `CASE`, `UNION ALL`) and ports to MySQL/PostgreSQL.

| Query | Answers | Output |
|---|---|---|
| Q1 | Loyal customers vs. promo-only buyers | `q1_loyalty_archetypes.csv` |
| Q2a–c | Behavioural patterns that predict value (tier profile, RFM, frequency) | `q2a/b/c_*.csv` |
| Q2d | Which seasons and categories attract low- vs. high-tenure customers | `q2d_season_category_tenure.csv` |
| Q2e | Frequency × tenure interaction | `q2e_frequency_tenure_interaction.csv` |
| Q3 | Geographies: organic demand vs. discount-driven volume | `q3_geographic_opportunity.csv` |
| Q4 / Q4b | Promo restructure matrix and CLV at stake per action | `q4_*.csv`, `q4b_*.csv` |
| Q5a–c | Ideal customer profile | `q5a/b/c_*.csv` |
| Master | Segment summary | `master_segment_summary.csv` |

Geography classification (Q3): 6 Organic Traction states, 9 Discount-Driven, 3 Under-Leveraged, 32 Developing. Thresholds are calibrated to the state-level distributions (CLV > 750, promo rate around the 43% median, loyal share > 11%).

### Dashboard (Power BI)

Four panels for a non-technical founding team: **customer pyramid** (value distribution by tier), **promo dependency vs. retention** by segment, **geographic opportunity map** by market type, and **category funnel** by purchase-history band. Data comes from the `pbi_*.csv` files.

---

## Known limitations

- **`clv_proxy` is a proxy, not true lifetime value.** With no timestamps it combines spend, frequency and purchase history; it cannot measure recency, tenure in months, or churn.
- **No cost data.** The Phase 3 recommendation depends on CAC, which the dataset does not contain. The playbook therefore specifies a CAC audit with a 1.5× CLV:CAC exit threshold rather than asserting those customers are unprofitable.
- **Value tiers are partly circular.** `value_tier` is built from `clv_proxy`, which includes `frequency_score` and `previous_purchases`, so "frequency predicts value" is partly mechanical. The 6.9× Weekly-vs-Annual gap is real in the data, but read it as a description of how the proxy behaves, not an independent test.
- **Hand-set thresholds.** Tenure bands, geography cut-offs and the 4.0 satisfaction threshold were chosen from the observed distributions, not tuned or validated out of sample.
- **The recommendations are untested hypotheses.** The 90-day pilot exists precisely because the data cannot show how promo-dependent customers will react to full prices.
- **Segment logic changed during the project.** An earlier `customer_segment` rule produced an empty "Discount Loyal" cell, because Def 2 excludes promo users by construction. It was replaced with the table above. `day1_feature_engineering.py` still contains the old rule; Day 2 overwrites it.

---

## Credits

Built for the Consulting & Analytics Club, IIT Guwahati, Summer Projects '26 (SQL | Consulting track).

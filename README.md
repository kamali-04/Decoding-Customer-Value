# Decoding Customer Value: A SQL-Driven Retention Strategy

A customer analytics project that uses Python and SQL to understand purchasing behaviour, estimate relative customer value, define loyalty segments, evaluate promotional dependence, and recommend more targeted retention strategies.

An additional **exploratory unsupervised-learning notebook** (`customer_clustering_kprototypes.ipynb`) extends the original rule-based segmentation with K-Prototypes clustering for mixed numerical and categorical customer data.

> **Scope note:** The original analysis is descriptive. The dataset is a one-row-per-customer snapshot, not a dated transaction history. The project does not establish future churn, retention, or causal promotion effects. The CLV measure is a constructed proxy, not realised or predicted lifetime revenue.

---

## Contents

- [Project overview](#project-overview)
- [Repository structure](#repository-structure)
- [Dataset](#dataset)
- [Python data preparation](#python-data-preparation)
- [SQL analysis](#sql-analysis)
- [Business recommendations](#business-recommendations)
- [Exploratory ML extension: K-Prototypes](#exploratory-ml-extension-k-prototypes)
- [Setup](#setup)
- [Run the project](#run-the-project)
- [Outputs](#outputs)
- [Important assumptions and limitations](#important-assumptions-and-limitations)

---

## Project overview

The project investigates a retail customer base and asks:

1. Which customer groups have stronger purchasing histories and higher estimated customer value?
2. How do loyalty, subscription status, promotional usage, purchase frequency, and satisfaction vary across customers?
3. Which geographic and seasonal/category combinations may offer business opportunities?
4. How could promotional spending be targeted rather than applied uniformly?
5. What behavioural characteristics describe the project's Ideal Customer Profile (ICP)?

The workflow is:

1. **Python / Pandas:** clean the customer data and create analytical features.
2. **SQLite / SQL:** aggregate customers, compare segments, and answer business questions.
3. **Executive Summary and Retention Playbook:** translate the findings into proposed business actions.
4. **K-Prototypes (exploratory extension):** discover mixed-data customer clusters and compare them with the existing rule-based segments.

## Repository structure

```text
Decoding-Customer-Value/
├── README.md
├── Dataset.csv
├── modified_dataset.csv
├── python_code.ipynb
├── sql.ipynb
├── customer_clustering_kprototypes.ipynb
├── Executive summary.pdf
├── playbook.docx
└── sql outputs/
    ├── customers_enriched.csv
    ├── master_segment_summary.csv
    ├── q1_loyalty_archetypes.csv
    ├── q2a_value_tier_profiles.csv
    ├── q2b_rfm_vs_value_tier.csv
    ├── q2c_frequency_vs_clv.csv
    ├── q2d_season_category_tenure.csv
    ├── q2e_frequency_tenure_interaction.csv
    ├── q3_geographic_opportunity.csv
    ├── q4_promo_restructure_matrix.csv
    ├── q4b_promo_action_summary.csv
    ├── q5a_icp_cohort_comparison.csv
    ├── q5b_icp_dimensional_profile.csv
    └── q5c_icp_pure_top_profiles.csv
```

The `sql outputs/` directory contains exported analysis results from the supplied project archive. The clustering notebook creates a separate `ml_clustering_outputs/` directory when it is run.

## Dataset

The supplied source file, `Dataset.csv`, contains **3,900 customer records and 18 original columns**. The columns describe customer demographics and purchase behaviour, including:

- Customer ID, age, and gender
- Item purchased and product category
- Purchase amount and location
- Size, colour, and season
- Review rating and subscription status
- Shipping type, discount applied, and promo code used
- Previous purchases, payment method, and purchase frequency

The supplied `modified_dataset.csv` contains **3,900 rows and 36 columns**: the original customer fields plus engineered features used by the SQL analysis.

## Python data preparation

Open `python_code.ipynb` to clean and enrich the original data.

### Cleaning

- Standardises column names to lowercase `snake_case`.
- Replaces missing `review_rating` values with the median. In the supplied run, **37 missing ratings** were imputed using a median of **3.8**.
- Checks that `discount_applied` and `promo_code_used` match for every row in the supplied dataset.
- Converts subscription, discount, and promo-code Yes/No fields into binary flags.
- Validates that no null values remain after the notebook's cleaning steps.

### Engineered features

| Feature | Description |
|---|---|
| `subscription_status_flag` | 1 for Yes, otherwise 0 |
| `discount_applied_flag` | 1 when a discount is applied, otherwise 0 |
| `promo_code_used_flag` | 1 when a promo code is used, otherwise 0 |
| `frequency_score` | Ordinal score from 1 (Annually) to 7 (Weekly) |
| `promo_dependency_score` | Uses `discount_applied_flag` as a promotional-use indicator |
| `satisfaction_flag` | 1 when `review_rating >= 4.0`, otherwise 0 |
| `clv_proxy` | `purchase_amount * frequency_score * log1p(previous_purchases)` |
| `value_tier` | Four quantile-based groups: Low, Mid, High, Premium |
| `age_group` | Predefined age bands |
| `r_score`, `f_score`, `m_score` | Quintile scores based on frequency score, previous purchases, and purchase amount |
| `rfm_total` | Sum of the three RFM component scores |
| `rfm_segment` | At-Risk, Needs Attention, Loyal, or Champions |
| `loyalty_def1` | Previous purchases at or above the median and an active subscription |
| `loyalty_def2` | Weekly/Bi-Weekly/Fortnightly purchase frequency, rating at least 4.0, and no promo-code use |
| `is_loyal` | Copies `loyalty_def2` |
| `customer_segment` | Rule-based segment: Brand Advocate, Subscription Member, Promo Hunter, or Passive Buyer |

### Customer segment assignment

The `customer_segment` rule is applied in this priority order:

1. `loyalty_def2 = 1` → **Brand Advocate**
2. Otherwise, subscription status is Yes → **Subscription Member**
3. Otherwise, promo dependency is 1 → **Promo Hunter**
4. Otherwise → **Passive Buyer**

The supplied enriched dataset has the following distribution:

| Customer segment | Customers | Share of base | Average CLV proxy |
|---|---:|---:|---:|
| Brand Advocate | 393 | 10.1% | $1,111.15 |
| Subscription Member | 1,053 | 27.0% | $733.13 |
| Promo Hunter | 624 | 16.0% | $694.69 |
| Passive Buyer | 1,830 | 46.9% | $629.36 |

**CLV proxy interpretation:** The formula is a relative customer-value score built from the available snapshot variables. It is not actual lifetime revenue, a discounted cash-flow CLV calculation, or a prediction of future customer spending.

## SQL analysis

Open `sql.ipynb`. It loads `modified_dataset.csv` into an **in-memory SQLite database** as a table named `customers`, then runs SQL queries and exports result tables.

| Analysis | Business question / method | Export |
|---|---|---|
| Q1 — Loyalty archetypes | Counts and value contribution for Organically Loyal, Subscription-Committed, Promo-Only Buyer, and Occasional Buyer | `q1_loyalty_archetypes.csv` |
| Q2a — Value-tier profiles | Compare frequency, purchase history, rating, subscription, promo use, and satisfaction across value tiers | `q2a_value_tier_profiles.csv` |
| Q2b — RFM vs value tier | Cross-tabulate RFM segments and CLV-based value tiers | `q2b_rfm_vs_value_tier.csv` |
| Q2c — Frequency vs CLV | Compare customer metrics across purchase-frequency categories | `q2c_frequency_vs_clv.csv` |
| Q2d — Season/category/tenure | Compare season and category groups by average previous purchases and other metrics | `q2d_season_category_tenure.csv` |
| Q2e — Frequency/tenure interaction | Compare purchase frequency and previous-purchase bands together | `q2e_frequency_tenure_interaction.csv` |
| Q3 — Geographic opportunity | Group by customer location and classify markets using CLV proxy, promo usage, and loyalty thresholds | `q3_geographic_opportunity.csv` |
| Q4 — Promotional restructuring matrix | Assign a proposed promotional action to each value-tier/customer-segment combination | `q4_promo_restructure_matrix.csv` |
| Q4b — Action summary | Summarise customer counts and CLV proxy by proposed promotional action | `q4b_promo_action_summary.csv` |
| Q5a — ICP cohort comparison | Compare all Premium customers with Premium Brand Advocates | `q5a_icp_cohort_comparison.csv` |
| Q5b — ICP dimensional profile | Compare Premium customers by gender, age group, category, frequency, payment method, and season | `q5b_icp_dimensional_profile.csv` |
| Q5c — Pure ICP profiles | Explore combinations of age group, gender, category, and season within the Pure ICP | `q5c_icp_pure_top_profiles.csv` |
| Master segment summary | Consolidated metrics by the four rule-based customer segments | `master_segment_summary.csv` |

The SQL notebook also exports an enriched customer table as `customers_enriched.csv`.

## Business recommendations

The Executive Summary and Retention Playbook translate the analysis into proposed actions.

### Promotional strategy

- **Premium Brand Advocates (205 customers):** the playbook recommends removing them from promotional targeting because this subgroup has high average CLV proxy and zero observed promo use. The proposed check is 30-day repeat purchase rate against baseline.
- **Premium and High promotional customers (835 customers):** run a 90-day controlled phase-out pilot with a holdout group. Track repeat purchase rate and average order value against the control group; review at Day 30, Day 60, and Day 90.
- **Low-tier promotional customers (409 customers):** audit acquisition cost and compare it with customer value before deciding whether to discontinue or cap discounts.
- **Mid-tier promotional customers (433 customers):** maintain and monitor because the analysis does not establish a clear case for immediate reduction.
- **Passive Buyers (1,830 customers):** the playbook identifies purchase-frequency activation, rather than discount removal, as the main opportunity for this group.

These are **recommendations**, not measured results from a completed promotional experiment.

### Ideal Customer Profile (ICP)

The project's Pure ICP is the **Premium + Brand Advocate** cohort (205 customers). The primary behavioural criteria are:

- Weekly or Bi-Weekly purchase frequency
- 30 or more previous purchases
- No promo-code usage
- Review rating of at least 4.0

In the supplied analysis, this cohort has an average CLV proxy of **$1,567.55**, average review rating of **4.47**, and zero observed promo-code usage. Demographic and product characteristics are treated as secondary lookalike-audience signals, not mandatory ICP filters.

## Exploratory ML extension: K-Prototypes

`customer_clustering_kprototypes.ipynb` adds an unsupervised clustering experiment. It uses **K-Prototypes** because the selected input contains both numerical and categorical variables.

### Model inputs

**Numerical:** `age`, `purchase_amount`, `previous_purchases`, `review_rating`.

**Categorical:** `gender`, `category`, `season`, `subscription_status`, `promo_code_used`, `frequency_of_purchases`, `payment_method`.

The notebook standardises numerical variables and leaves categorical values as categorical strings for K-Prototypes. It intentionally excludes customer IDs and existing rule-based labels/scores from model inputs. The CLV proxy and existing segments are used only when profiling and comparing clusters after fitting.

### Workflow

1. Load and validate `modified_dataset.csv`.
2. Prepare numerical and categorical features, handling missing values defensively.
3. Standardise numerical inputs.
4. Fit candidate models for K = 2 through 8 and inspect the cost/elbow curve.
5. Choose `SELECTED_K` after considering the elbow, cluster sizes, stability, and business interpretability. The notebook's initial value of 4 is a trial value, not a validated optimum.
6. Profile clusters and compare them with the existing rule-based segments and value tiers.
7. Save the clustered data, profiles, comparison table, and model bundle.

### Clustering outputs

After the clustering notebook runs, it creates:

```text
ml_clustering_outputs/
├── customers_with_ml_clusters.csv
├── ml_cluster_profiles.csv
├── ml_cluster_vs_rule_segments.csv
└── kprototypes_customer_segmentation.joblib
```

The cluster IDs have no inherent business meaning. Name clusters only after inspecting their profiles. The cross-tabulation against existing rule-based segments is a comparison, **not a model accuracy score**.

This is exploratory segmentation, not churn prediction or proof of future retention. No clustering result or best K is claimed in this README because the new notebook has not yet been executed and reviewed.

## Setup

The supplied archive does not include a `requirements.txt`. On macOS, create and activate a virtual environment from the project root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pandas numpy jupyter notebook matplotlib scikit-learn joblib kmodes
```

`sqlite3` is part of Python's standard library and does not need a separate pip installation.

## Run the project

### 1. Prepare the enriched dataset

Open `python_code.ipynb` and update its hard-coded `RAW_PATH` and `OUT_PATH` values to point to files in your project folder. For example:

```python
from pathlib import Path

PROJECT_DIR = Path.cwd()
RAW_PATH = PROJECT_DIR / "Dataset.csv"
OUT_PATH = PROJECT_DIR / "modified_dataset.csv"
```

Run the notebook from top to bottom. The supplied Python notebook currently sets `OUT_PATH` to `customers_enriched.csv`, while the supplied SQL and clustering notebooks expect `modified_dataset.csv`. Either change the Python notebook's output path as shown above, or update the downstream input paths to match the file you generate.

The archive already contains `modified_dataset.csv`, so you can use that supplied enriched file without rerunning the Python notebook.

### 2. Run the SQL analysis

Open `sql.ipynb`. Update `INPUT_PATH` to the location of `modified_dataset.csv` and set `OUTPUT_DIR` to the folder where you want the exported query results. For example:

```python
from pathlib import Path

PROJECT_DIR = Path.cwd()
INPUT_PATH = PROJECT_DIR / "modified_dataset.csv"
OUTPUT_DIR = PROJECT_DIR / "sql outputs"
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)
```

Run the notebook from top to bottom. It creates an in-memory SQLite database, loads the `customers` table, executes the analysis queries, exports their results, and closes the connection.

### 3. Run the K-Prototypes clustering extension

Place `customer_clustering_kprototypes.ipynb` in the project root beside `modified_dataset.csv`. Launch Jupyter from that directory:

```bash
jupyter notebook
```

Open the clustering notebook and run it from top to bottom. Inspect the elbow curve before confirming or changing `SELECTED_K` in the model-fitting cell. Review the cluster profiles before assigning business labels.

## Important assumptions and limitations

1. **Snapshot data:** There is one record per customer, not a dated sequence of transactions. The project cannot directly calculate true recency, customer retention over time, or future churn.
2. **RFM terminology:** The notebook's `r_score` is based on `frequency_score`, not the date of the customer's most recent purchase. It is therefore a frequency-based proxy rather than conventional recency.
3. **CLV proxy:** `clv_proxy` is constructed from purchase amount, an ordinal frequency score, and previous purchases. It is not actual lifetime revenue or a validated future-value prediction.
4. **Duplicate promotion fields:** In the supplied dataset, `discount_applied` and `promo_code_used` are identical for all 3,900 rows. They should not be treated as independent behavioural signals.
5. **Subscription/promotion association:** The supplied analysis finds 100% promo-code use among Subscription Members. This is an observed property of this dataset, not a general claim about subscriptions.
6. **Loyalty definitions:** The loyalty categories are operational rules created for this analysis, not independently validated ground-truth labels.
7. **Descriptive, not causal:** Observed associations do not establish that promotions cause higher or lower customer value. The proposed 90-day pilot requires a properly assigned treatment and control group.
8. **Geographic thresholds:** The Q3 market types use manually specified thresholds and a minimum group size of 50 customers. They are business heuristics, not learned predictions.
9. **ML cluster interpretation:** K-Prototypes results depend on feature selection, scaling, the categorical/numerical balance (`gamma`), initialisation, and K. Evaluate cluster sizes and stability before using them for decisions.
10. **Small subgroup profiles:** Q5c includes groups with as few as two customers. Treat those combinations as exploratory, not statistically robust targeting rules.

---

## Project summary

This project combines data cleaning, feature engineering, SQL-based customer analysis, and business recommendations. The K-Prototypes notebook adds an exploratory ML segmentation layer while keeping the original rule-based workflow intact. The next step toward predictive retention would be to obtain dated customer transaction history and outcomes, then build and validate a separate churn or customer-value prediction model using a time-aware evaluation design.

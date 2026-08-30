# Apex Dynamics Executive Data Hygiene Audit

**Corporate visualization theme:** `#2563EB`  
**Pipeline scope:** full comprehensive analysis across data hygiene, financial integrity, product performance, returns, time series, discount/revenue relationships, and data-quality intervention priorities.

## Executive Summary

The Apex Dynamics source file required structural and analytical remediation before it could be used safely for reporting. The raw CSV contained unquoted thousands separators in `gross_rev`, which created delimiter collisions; inconsistent categorical casing; a repeated `order_id`; invalid calendar values; standardized missing-value tokens; negative financial values; and a missing revenue observation.

The pipeline preserves economically meaningful anomalies instead of silently deleting them. Negative revenue remains in the analytical dataset and is flagged as a potential refund or chargeback. Invalid dates are converted to `NaT` rather than guessed. Redundant duplicates are logged before removal. Statistical revenue extremes are identified using the resistant 1.5×IQR method and remain available for executive review.

### Executive KPIs

| KPI | Value |
| --- | --- |
| Raw source rows | 10 |
| Clean rows | 9 |
| Source records affected by hygiene flags | 8 (80.00%) |
| Redundant duplicate rows removed | 1 |
| Gross revenue before duplicate isolation | $1,205,151.75 |
| Gross revenue after duplicate isolation | $1,079,751.25 |
| Invalid dates | 2 |
| Negative-revenue records | 2 |
| Missing-revenue records | 1 |
| Missing-discount records | 3 |
| Missing return-status records | 1 |
| Revenue IQR lower bound | $-176,850.75 |
| Revenue IQR upper bound | $306,751.25 |

## 1. Data Hygiene Audit

### Source-file structural issue

The input is intended to contain eight columns, but values such as `$125,400.50` and `4,500.00` contain unquoted delimiter commas. The pipeline repairs the malformed revenue field using the stable surrounding schema before parsing the source into Pandas.

### Corrections and safeguards

- Column names are normalized to `snake_case`.
- Identifier fields are retained as strings.
- Standard missing tokens (`N/A`, `NULL`, `missing`, etc.) are converted to explicit missing values.
- Product categories, region codes, and return statuses are stripped and lowercased to prevent phantom reporting groups.
- Currency symbols and thousands separators are removed before numeric conversion.
- Percent signs are removed and discounts are stored as percentage points.
- Impossible and unparseable dates are coerced to `NaT`.
- Negative revenue is retained and flagged instead of automatically deleted.
- `order_id` collisions are logged before duplicate isolation.
- IQR outliers are flagged instead of silently excluded.

### Missing-value percentages after redundant duplicate removal

| column | missing_count | missing_pct |
| --- | --- | --- |
| discount_pct | 3 | 33.33 |
| discount_pct_raw | 3 | 33.33 |
| transaction_date | 2 | 22.22 |
| transaction_month | 2 | 22.22 |
| gross_rev | 1 | 11.11 |
| gross_rev_raw | 1 | 11.11 |
| is_returned | 1 | 11.11 |
| negative_revenue_flag | 1 | 11.11 |
| net_positive_revenue_flag | 1 | 11.11 |
| returned_numeric | 1 | 11.11 |

### Records requiring review

| order_id | customer_id | transaction_date_raw | product_category | region_code | gross_rev | discount_pct | is_returned | negative_revenue_flag | missing_revenue_flag | missing_discount_flag | missing_return_status_flag | invalid_return_status_flag | invalid_date_flag | order_id_collision_flag | exact_duplicate_flag | revenue_outlier_flag |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ORD-9001 | CUST_401 | 2026-01-15 | enterprise cloud | na | $125,400.50 | 15.0% | no | False | False | False | False | False | False | True | False | False |
| ORD-9002 | CUST_402 | 2026-02-30 | consumer hardware | emea | $4,500.00 | 0.0% |  | False | False | False | True | False | True | False | False | False |
| ORD-9001 | CUST_401 | 2026-01-15 | enterprise cloud | na | $125,400.50 | 15.0% | no | False | False | False | False | False | False | True | True | False |
| ORD-9004 | CUST_404 | 2026-03-10 | consumer hardware | apac | $-1,250.00 | missing | yes | True | False | True | False | False | False | False | False | False |
| ORD-9005 | CUST_405 | invalid_date | enterprise cloud | latam | $340,000.00 | 25.0% | no | False | False | False | False | False | True | False | False | True |
| ORD-9006 | CUST_406 | 2026-04-18 | smart home iot | emea | missing | missing | no |  | True | True | False | False | False | False | False | False |
| ORD-9009 | CUST_409 | 2026-06-14 | consumer hardware | emea | $-450.00 | missing | yes | True | False | True | False | False | False | False | False | False |
| ORD-9010 | CUST_410 | 2026-06-30 | enterprise cloud | latam | $510,000.00 | 20.0% | no | False | False | False | False | False | False | False | False | True |

## 2. Financial Integrity

Gross financial volume before redundant duplicate isolation was **$1,205,151.75**. After removing only confirmed redundant normalized duplicates, gross financial volume was **$1,079,751.25**.

The difference is attributable to duplicate isolation rather than a blanket outlier-removal policy. Negative revenue remains in the clean dataset because it can reflect legitimate returns, refunds, or chargebacks.

Two cleaned records combine a negative `gross_rev` value with `is_returned = yes` in **2** case(s). This alignment is operationally plausible and should be reconciled against Apex Dynamics' return/refund rules rather than discarded.

## 3. Pattern Discovery

### Product-category revenue concentration

The largest aggregate category in this sample is **enterprise cloud**, with observed gross revenue of **$1,064,600.50** after redundant duplicate isolation.

![Gross revenue by product category](images/revenue_by_product_category.png)

**Why this chart:** A bar chart provides a direct categorical comparison and makes concentration by product family immediately visible.

### Revenue dispersion and unusual transactions

The revenue field spans negative transactions through very large positive transactions. The boxplot emphasizes skew, central spread, and extreme values without assuming that large enterprise transactions are errors.

![Transaction revenue distribution](images/revenue_distribution_boxplot.png)

**Why this chart:** A boxplot is appropriate for a skewed continuous financial variable and pairs naturally with IQR-based anomaly detection.

### Return concentration by region

![Return status by region](images/returns_by_region.png)

**Why this chart:** Grouped categorical bars make it easy to compare returned, non-returned, and missing-status transactions across regions.

### Revenue over time

![Monthly gross revenue](images/monthly_gross_revenue.png)

**Why this chart:** A line chart is appropriate for ordered time-series data. Records with invalid dates are excluded from temporal placement rather than assigned fabricated dates.

### Discount vs. transaction value

![Discount vs. gross revenue](images/discount_vs_gross_revenue.png)

The observed Pearson correlation among records with both fields present is **0.847** if enough observations are available. Because this is a very small portfolio sample with missing values and a highly dispersed revenue field, the statistic should be treated as exploratory rather than causal evidence.

**Why this chart:** A scatterplot exposes the relationship, clusters, and influential transactions more transparently than a single correlation coefficient.

### Executive data-quality intervention priorities

![Data quality flags](images/data_quality_flags.png)

**Why this chart:** A ranked quality-flag view translates technical audit conditions into a concrete remediation queue for Finance, Sales Operations, and Data Engineering.

## 4. Recommended Business Focus Areas

### Finance
Reconcile negative revenue with return/refund events, recover the missing revenue transaction from the system of record, and ensure duplicate `order_id` controls exist upstream so revenue cannot be double counted.

### Sales
Use the normalized product and regional fields for performance reporting, but keep high-value enterprise transactions visible rather than automatically treating them as erroneous outliers.

### Marketing
Treat the discount/revenue relationship as exploratory in this small sample. A larger historical dataset is required before discount strategy should be inferred from correlation.

### Operations
Investigate invalid transaction dates and the missing return status. These are source-data quality issues that cannot be repaired truthfully through statistical imputation.

### Data Engineering
Enforce properly quoted CSV exports or, preferably, a typed interchange format. Add schema tests for column count, primary-key uniqueness, date validity, numeric parseability, and standardized missing tokens before downstream ingestion.

## 5. Methodology Justification

### Why invalid dates become `NaT`
An impossible date such as `2026-02-30` has no defensible replacement without source evidence. `NaT` preserves the fact that the time value is unknown while allowing the rest of the observation to remain available.

### Why negative revenue is not deleted
Negative financial observations may be legitimate refunds, chargebacks, credits, or reversals. Automatic deletion would create a risk of overstating revenue.

### Why IQR is used for outlier detection
The revenue distribution is highly dispersed. The interquartile range uses the middle 50% of observations and is resistant to extreme transactions, making it more appropriate than mean/standard-deviation rules for skewed financial data.

### Why missing financial values are not imputed automatically
For transactional revenue, an invented mean or median can directly distort financial reporting. The preferred production action is recovery from the source system. Statistical imputation should only be introduced for a clearly defined downstream analytical use case with explicit governance.

### Why duplicate records are logged before removal
An audit trail must distinguish a source-system collision from a legitimate business event. The pipeline therefore records duplicate flags first and removes only a confirmed redundant normalized observation.

## 6. Cross-Departmental Glossary

| Term | Plain-English meaning |
| --- | --- |
| `snake_case` | A consistent naming style such as `gross_rev`, which prevents the same concept from appearing under multiple header formats. |
| Vectorization | Processing whole columns in Pandas at once instead of manually stepping through each transaction. |
| Coercion to `NaT` | Converting an invalid date to an explicit “not a valid time” marker rather than guessing a replacement date. |
| Missing value | A field whose real business value is unavailable or not supplied. |
| Imputation | Filling a missing value with an estimated value such as a median. |
| IQR | The spread of the middle 50% of observations; a robust way to measure dispersion when extreme values exist. |
| Outlier | A value far from the typical range. It can be an error or a legitimate unusual transaction. |
| Primary-key collision | Two source records use the same supposedly unique identifier. |
| EDA | Exploratory Data Analysis: the initial statistical and visual investigation of a dataset. |
| `pd.to_numeric()` | Converts text-formatted numbers into values that can be calculated. |
| `pd.to_datetime()` | Converts text into validated dates. |
| `errors='coerce'` | Tells Pandas to mark an invalid conversion as missing instead of guessing or stopping the pipeline. |
| `.groupby()` | Organizes rows into business groups such as product categories or regions before calculating summaries. |
| `.describe()` | Produces a compact statistical profile of a dataset. |

## 7. Repository Artifacts

The pipeline writes:

- `apex_dynamics_clean_transactions.csv` — analysis-ready transaction data.
- `apex_dynamics_audit_records.csv` — source-level audit flags and preserved raw values.
- `apex_dynamics_audit_metrics.csv` — executive KPI summary.
- `apex_dynamics_missing_values.csv` — missing-value counts and percentages.
- `images/*.png` — publication-ready branded charts.
- `apex_dynamics_executive_audit_report.md` — this executive report.

## Governance Note

This sample is sufficiently small that all analytical relationships should be presented as descriptive and exploratory. No causal conclusion should be drawn from category, discount, regional, or return patterns without a larger and more representative historical dataset.

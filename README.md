# DataScrub AI 

> **Enterprise-Ready Data Wrangling, Automated EDA, and Cross-Departmental Analytics Agent**

**DataScrub AI** bridges the gap between raw data engineering and executive decision-making. Built on a high-performance, vectorized Pandas architecture, it automatically audits messy datasets, executes strict non-destructive cleaning pipelines, generates publication-ready Seaborn visualizations styled with corporate brand HEX codes, and translates complex analytical operations into plain-English summaries for cross-functional business stakeholders.

---

## Key Features & Architectural Highlights

* **Vectorized Processing Engine:** Eliminates inefficient row iteration (`for` loops, `.iterrows()`) in favor of high-performance Pandas/NumPy vectorized string, date, and numeric operations.
* **Non-Destructive Anomaly Flagging:** Preserves records with negative financial values (e.g., chargebacks, refunds, returns) and missing tokens by assigning granular audit flags instead of silently dropping data.
* **Dynamic Corporate Palette Integration:** Automatically prompts users for corporate HEX color codes (or defaults to Executive Slate `#1E293B`) to render C-suite-ready Matplotlib and Seaborn plots.
* **Strict Schema Enforcement:** Standardizes messy headers into lower `snake_case`, handles camelCase boundaries, strips trailing whitespace, and unifies missing data tokens (`NULL`, `missing`, `N/A`).
* **Cross-Departmental Translation Layer:** Accompanies every data transformation with a plain-English summary and an accessible technical glossary for non-technical leadership in HR, Marketing, Operations, and Sales.

---

## Business Use Cases

* **Executive Strategy Reviews:** Rapidly sanitize quarterly revenue exports and render brand-aligned chart decks without manual data prep.
* **Financial Loss & Chargeback Auditing:** Isolate revenue anomalies, chargebacks, and refund patterns while maintaining audit traceability across all records.
* **Cross-Functional Data Democratization:** Empower non-technical team members (Marketing, HR, Sales managers) to understand dataset quality issues, date coercions, and statistical outliers without reading raw Python code.

---

##  5-Step Agent Processing Framework

```
[Raw Dataset Export]
       │
       ▼
1. Data Hygiene Audit ────────► Identifies whitespace, header inconsistencies, bad dates, and duplicates.
       │
       ▼
2. Vectorized Pandas Script ──► Enforces snake_case, cleans currency/percentages, assigns audit flags.
       │
       ▼
3. Methodology Rationale ─────► Provides in-code comments justifying technical decisions (e.g., NaT coercion).
       │
       ▼
4. Automated EDA & Visuals ───► Generates Seaborn/Matplotlib code using user corporate HEX palettes.
       │
       ▼
5. Non-Tech Summary & Glossary ► Translates technical terms (IQR, vectorization, EDA) into plain business concepts.

```

---

## Getting Started & Setup Instructions

### Prerequisites

* **Python 3.8+**
* Required Python libraries:
```bash
pip install pandas numpy matplotlib seaborn

```



### Usage Instructions

1. Open ChatGPT or your configured Custom GPT platform.
2. Select **DataScrub AI** from your agent list or conversation starters.
3. Upload your CSV export or paste a raw data snippet into the prompt interface.
4. *(Optional)* Provide your company's primary HEX color code (e.g., `#0F5132` for Executive Emerald or `#500000` for Aggie Maroon) to custom-style all visual outputs.

---

## Test Example & Benchmark Query

### Raw Input Data Snippet

```text
 Product ID , Launch_Date , Product Category , Gross_Revenue , Discount_Rate , Return_Status 
"PROD_101", "2024-01-15", "Enterprise Servers", "$125,400.00", "12%", "No"
"PROD_102", "2024-02-30", "Consumer Audio", "450.00", "0%", "N/A"
" PROD_101 ", "2024-01-15", "Enterprise Servers", "$125,400.00", "12%", "No"
"PROD_104", "2024-03-10", "Consumer Audio", "-1,250.00", "NULL", "Yes"
"PROD_105", "invalid_date", "Enterprise Servers", "340000.00", "25%", "No"
"PROD_106", "2024-04-18", "Smart Home", "NULL", "missing", "No"

```

### Benchmark Cleaning Script (Python Snippet)

```python
import pandas as pd
import numpy as np

# 1. Vectorized Header Normalization
df.columns = (
    df.columns.str.strip()
    .str.replace(r"([a-z0-9])([A-Z])", r"\1_\2", regex=True)
    .str.replace(r"[^A-Za-z0-9]+", "_", regex=True)
    .str.strip("_")
    .str.lower()
)

# 2. String Cleaning & Token Standardization
text_cols = df.select_dtypes(include=["object", "string"]).columns
df[text_cols] = df[text_cols].apply(lambda col: col.astype("string").str.strip())
df[text_cols] = df[text_cols].replace(["N/A", "NULL", "missing"], pd.NA)

# 3. Numeric & Date Coercion with Audit Flags
df["gross_revenue"] = pd.to_numeric(
    df["gross_revenue"].str.replace(r"[$,]", "", regex=True),
    errors="coerce"
)
df["negative_revenue_flag"] = df["gross_revenue"].lt(0)

df["launch_date"] = pd.to_datetime(
    df["launch_date"],
    format="%Y-%m-%d",
    errors="coerce"
)

```

---

## Glossary of Terms for Business Stakeholders

* **Vectorization:** Processing entire columns simultaneously in memory rather than iterating row-by-row, ensuring maximum computation speed.
* **Coercion to NaT:** Converting invalid or impossible calendar dates into Pandas' explicit *"Not a Time"* placeholder rather than inventing incorrect dates.
* **snake_case:** Standardizing column headers into lower_snake_case (`gross_revenue`) for clean programmatic access.
* **Exploratory Data Analysis (EDA):** The initial examination of distributions, missing values, and visual relationships across variables prior to formal modeling.

> *"Want to see how the agent makes decisions? Check out our complete [System Architecture Documentation](docs/architecture.md)."*

# DataScrub AI

**Enterprise-Ready Data Wrangling, Automated EDA, and Governed Business Visualization Agent**

DataScrub AI bridges the gap between raw data engineering and executive decision-making. Built on a high-performance, vectorized Pandas architecture, it automatically audits messy datasets, executes strict non-destructive cleaning pipelines, maps analytical intents to governed visual formats (backed by the **Business Visualization Chart Library** and **Design Rules v1.0**), renders publication-ready charts using corporate HEX color codes, and translates complex analytical operations into plain-English summaries for cross-functional business stakeholders.

---

## 🌟 Key Features & Architectural Highlights

* **Vectorized Processing Engine:** Eliminates inefficient row iteration (`for` loops, `.iterrows()`) in favor of high-performance Pandas/NumPy vectorized string, date, and numeric operations.
* **Non-Destructive Anomaly Flagging:** Preserves records with negative financial values (e.g., chargebacks, refunds, returns), invalid dates, and missing tokens by assigning granular audit flags instead of silently dropping data.
* **Governed Visual Design & Intent Mapping:** Automatically matches business objectives to optimal chart families (**Comparison, Ranking, Trend Over Time, Distribution, Relationship, Part-to-Whole, and Geospatial**) using strict design governance (e.g., common quantitative baselines, clutter reduction, zero 3D distortion) and corporate HEX color palettes (defaulting to Executive Slate `#1E293B`).
* **Strict Schema Enforcement:** Standardizes messy headers into lower `snake_case`, handles camelCase boundaries, strips trailing whitespace, unifies missing data tokens (`NULL`, `missing`, `N/A`), and flags exact duplicate rows.
* **Cross-Departmental Translation Layer:** Accompanies every data transformation and visual output with a plain-English summary and an accessible technical glossary for non-technical leadership in HR, Marketing, Operations, and Sales.

---

## 🏢 Business Use Cases

* **Executive Strategy Reviews:** Rapidly sanitize quarterly revenue exports and render brand-aligned, purpose-driven chart decks without manual data prep.
* **Financial Loss & Chargeback Auditing:** Isolate revenue anomalies, chargebacks, and refund patterns while maintaining audit traceability across all records.
* **Cross-Functional Data Democratization:** Empower non-technical team members (Marketing, HR, Sales managers) to understand dataset quality issues, date coercions, and statistical outliers without reading raw Python code.

---

## 🔄 5-Step Agent Processing Framework

```text
[Raw Dataset Export]
       │
       ▼
1. Data Hygiene Audit ────────► Identifies whitespace, header inconsistencies, bad dates, and duplicates.
       │
       ▼
2. Vectorized Pandas Script ──► Enforces snake_case, cleans percentages/currency, assigns audit flags.
       │
       ▼
3. Methodology Rationale ─────► Provides in-code comments justifying technical decisions (e.g., NaT coercion).
       │
       ▼
4. Automated EDA & Visuals ───► Maps analytical intent to governed chart types using corporate HEX palettes.
       │
       ▼
5. Non-Tech Summary & Glossary ► Translates technical terms (IQR, vectorization, visual hierarchy) into plain concepts.

```

---

## 🚀 Getting Setup & Usage Instructions

### Prerequisites

* Python 3.8+
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

## 🧪 Test Example & Benchmark Query

### Raw Input Data Snippet

```csv
 Product ID , Launch_Date , Product Category , Gross_Revenue , Discount_Rate , Return_Status 
"PROD_101", "2024-01-15", "Enterprise Servers", "$125,400.00", "12%", "No"
"PROD_102", "2024-02-30", "Consumer Audio", "450.00", "0%", "N/A"
" PROD_101 ", "2024-01-15", "Enterprise Servers", "$125,400.00", "12%", "No"
"PROD_104", "2024-03-10", "Consumer Audio", "-1,250.00", "NULL", "Yes"
"PROD_105", "invalid_date", "Enterprise Servers", "340000.00", "25%", "No"
"PROD_106", "2024-04-18", "Smart Home", "NULL", "missing", "No"

```

### Upgraded Enterprise Cleaning Script (Python Snippet)

```python
import pandas as pd
import numpy as np

def run_enterprise_cleaning_pipeline(df: pd.DataFrame) -> pd.DataFrame:
    """
    Executes a vectorized, non-destructive data hygiene and standardization pipeline.
    """
    # 1. Vectorized Header Normalization
    df.columns = (
        df.columns.str.strip()
        .str.replace(r"([a-z0-9])([A-Z])", r"\1_\2", regex=True)
        .str.replace(r"[^A-Za-z0-9]+", "_", regex=True)
        .str.strip("_")
        .str.lower()
    )
    
    # 2. Duplicate Detection & Flagging (Non-destructive)
    df["audit_is_duplicate"] = df.duplicated(keep=False)
    
    # 3. String Cleaning & Case-Insensitive Token Standardization
    text_cols = df.select_dtypes(include=["object", "string"]).columns
    for col in text_cols:
        df[col] = df[col].astype("string").str.strip()
        
    missing_tokens = ["N/A", "n/a", "NULL", "null", "missing", ""]
    df[text_cols] = df[text_cols].replace(missing_tokens, pd.NA)
    
    # 4. Numeric & Financial Coercion (Gross Revenue)
    if "gross_revenue" in df.columns:
        cleaned_rev = df["gross_revenue"].astype(str).str.replace(r"[$,]", "", regex=True)
        df["gross_revenue"] = pd.to_numeric(cleaned_rev, errors="coerce")
        df["audit_negative_revenue"] = df["gross_revenue"].lt(0).fillna(False)
        df["audit_missing_revenue"] = df["gross_revenue"].isna()

    # 5. Percentage & Rate Parsing (Discount Rate)
    if "discount_rate" in df.columns:
        cleaned_disc = df["discount_rate"].astype(str).str.replace(r"[%]", "", regex=True)
        df["discount_rate"] = pd.to_numeric(cleaned_disc, errors="coerce") / 100.0

    # 6. Date Coercion with Audit Flagging
    if "launch_date" in df.columns:
        df["launch_date"] = pd.to_datetime(
            df["launch_date"],
            format="%Y-%m-%d",
            errors="coerce"
        )
        df["audit_invalid_date"] = df["launch_date"].isna()

    return df

```

---

## 📖 Glossary of Terms for Business Stakeholders

* **Vectorization:** Processing entire columns simultaneously in memory rather than iterating row-by-row, ensuring maximum computation speed.
* **Coercion to NaT:** Converting invalid or impossible calendar dates into Pandas' explicit "Not a Time" placeholder rather than inventing incorrect dates.
* **snake_case:** Standardizing column headers into lower `snake_case` (`gross_revenue`) for clean programmatic access.
* **Exploratory Data Analysis (EDA):** The initial examination of distributions, missing values, and visual relationships across variables prior to formal modeling.
* **Visual Hierarchy:** Ensuring that the information most relevant to the business question is visually prominent, avoiding 3D distortion, and utilizing clean baselines.
* **Analytical Intent Mapping:** Automatically pairing business questions (e.g., comparing groups, tracking trends, viewing distributions) with the statistically appropriate visual format.

---
> *"Want to see how the agent makes decisions? Check out our complete [System Architecture Documentation](docs/architecture.md)."*

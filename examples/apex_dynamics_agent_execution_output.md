I’ll process the Apex Dynamics export through five deliverables: (1) Data Hygiene Audit, (2) executable Pandas cleaning script, (3) methodology justification, (4) pattern discovery + executive visualization recommendations, and (5) a cross-departmental summary/glossary. No corporate color was specified, so the visualization code below uses the default Executive Slate #1E293B theme; send me your primary corporate HEX code or brand color if you want it substituted.

I’m also applying the supplied data-dictionary rules: standard missing tokens include N/A, NULL, and missing; invalid calendar dates should be coerced to NaT; negative financial values should be flagged rather than automatically deleted; and the required column convention is snake_case.

1. Data Hygiene Audit

The six-row export contains several deliberate data-quality problems that would distort an executive analysis if used as-is.

Issue	Finding	Treatment
Header formatting	Headers contain leading/trailing spaces and inconsistent capitalization	Convert to snake_case
Product key padding	" PROD_101 " contains spaces	Strip whitespace
Duplicate entity	PROD_101 occurs twice after trimming	Keep one record; flag/remove exact duplicate
Invalid date	2024-02-30 does not exist	Coerce to NaT
Invalid date token	invalid_date	Coerce to NaT
Currency formatting	$125,400.00 contains $ and commas	Convert to numeric 125400.00
Negative revenue	PROD_104 = -1250.00	Preserve and flag as potential refund/chargeback
Missing revenue	PROD_106 = NULL	Convert to NaN; don't invent revenue
Percentage formatting	12%, 0%, 25%	Convert to decimal rates 0.12, 0.00, 0.25
Missing discount	NULL, missing	Standardize as NaN
Missing return status	N/A	Standardize as missing

After normalization, there are 5 unique products rather than 6 source rows. Two of the five unique products have unusable launch dates (40%), one has missing revenue (20%), two have missing discount rates (40%), and one has missing return status (20%).

The four known revenue values are $125,400, $450, −$1,250, and $340,000, totaling $464,600. PROD_105 contributes about 73.2% of known gross revenue, making revenue highly concentrated. PROD_104's negative value should not be treated as ordinary negative sales—it should remain visible for investigation under the supplied refund/chargeback rule.

Using the supplied 1.5× IQR anomaly rule, none of the four known revenue observations falls outside the calculated IQR fences. With only four observed revenue values, however, statistical outlier conclusions should be treated cautiously.

2. Executable Pandas Cleaning Script
import pandas as pd
import numpy as np
import re
from io import StringIO

raw_csv = '''
 Product ID , Launch_Date , Product Category , Gross_Revenue , Discount_Rate , Return_Status
"PROD_101","2024-01-15","Enterprise Servers","$125,400.00","12%","No"
"PROD_102","2024-02-30","Consumer Audio","450.00","0%","N/A"
" PROD_101 ","2024-01-15","Enterprise Servers","$125,400.00","12%","No"
"PROD_104","2024-03-10","Consumer Audio","-1,250.00","NULL","Yes"
"PROD_105","invalid_date","Enterprise Servers","340000.00","25%","No"
"PROD_106","2024-04-18","Smart Home","NULL","missing","No"
'''

df = pd.read_csv(StringIO(raw_csv), skipinitialspace=True)

# ---------------------------------------------------------
# 1. STRICT SCHEMA ENFORCEMENT
# ---------------------------------------------------------

# Vectorized header normalization:
# - remove outer whitespace
# - separate camelCase boundaries if encountered
# - replace non-alphanumeric characters with underscores
# - enforce lowercase snake_case
headers = (
    df.columns.to_series()
      .str.strip()
      .str.replace(r"([a-z0-9])([A-Z])", r"\1_\2", regex=True)
      .str.replace(r"[^A-Za-z0-9]+", "_", regex=True)
      .str.strip("_")
      .str.lower()
)

df.columns = headers

# Standard missing-value tokens supplied by the data dictionary.
missing_tokens = [
    "N/A", "null", "NULL", "None", "", "NaN", "missing"
]

# Normalize whitespace in text columns without row iteration.
text_cols = df.select_dtypes(include=["object", "string"]).columns
df[text_cols] = df[text_cols].apply(
    lambda col: col.astype("string").str.strip()
)

df[text_cols] = df[text_cols].replace(missing_tokens, pd.NA)


# ---------------------------------------------------------
# 2. AUDIT FLAGS BEFORE DE-DUPLICATION
# ---------------------------------------------------------

# Duplicate IDs are flagged after whitespace normalization so
# "PROD_101" and " PROD_101 " resolve to the same business key.
df["duplicate_product_id_flag"] = df["product_id"].duplicated(
    keep=False
)

# Keep the first occurrence when duplicate records represent
# the same product export. The audit flag preserves traceability.
df = df.drop_duplicates(subset=["product_id"], keep="first").copy()


# ---------------------------------------------------------
# 3. DATE CLEANING
# ---------------------------------------------------------

# Invalid calendar dates such as 2024-02-30 and arbitrary
# strings such as "invalid_date" become NaT rather than being
# guessed. This protects time-series analysis from fake dates.
df["launch_date_raw"] = df["launch_date"]

df["launch_date"] = pd.to_datetime(
    df["launch_date"],
    format="%Y-%m-%d",
    errors="coerce"
)

df["invalid_launch_date_flag"] = (
    df["launch_date_raw"].notna() &
    df["launch_date"].isna()
)


# ---------------------------------------------------------
# 4. FINANCIAL CLEANING
# ---------------------------------------------------------

# Remove currency symbols and thousands separators before
# numeric conversion. Bad/non-numeric values become NaN.
df["gross_revenue"] = pd.to_numeric(
    df["gross_revenue"]
      .str.replace(r"[$,]", "", regex=True),
    errors="coerce"
)

# Negative financial values are retained because they can
# represent legitimate refunds, returns, or chargebacks.
df["negative_revenue_flag"] = df["gross_revenue"].lt(0)

# Explicitly identify missing revenue rather than imputing it.
# Revenue should not be invented without a defensible business rule.
df["missing_revenue_flag"] = df["gross_revenue"].isna()


# ---------------------------------------------------------
# 5. PERCENTAGE CLEANING
# ---------------------------------------------------------

# Convert percentage strings into decimal numeric rates:
# "12%" -> 0.12, "25%" -> 0.25.
df["discount_rate"] = (
    pd.to_numeric(
        df["discount_rate"].str.replace("%", "", regex=False),
        errors="coerce"
    )
    .div(100)
)

df["missing_discount_flag"] = df["discount_rate"].isna()


# ---------------------------------------------------------
# 6. RETURN-STATUS QUALITY FLAG
# ---------------------------------------------------------

df["missing_return_status_flag"] = df["return_status"].isna()


# ---------------------------------------------------------
# 7. IQR REVENUE ANOMALY DETECTION
# ---------------------------------------------------------

# The supplied data dictionary defines 1.5 × IQR as the
# anomaly threshold. Values are flagged, never automatically
# deleted, so executives retain visibility into unusual sales.
q1 = df["gross_revenue"].quantile(0.25)
q3 = df["gross_revenue"].quantile(0.75)
iqr = q3 - q1

lower_bound = q1 - (1.5 * iqr)
upper_bound = q3 + (1.5 * iqr)

df["revenue_iqr_outlier_flag"] = (
    df["gross_revenue"].lt(lower_bound) |
    df["gross_revenue"].gt(upper_bound)
)


# ---------------------------------------------------------
# 8. CONSOLIDATED DATA-QUALITY FLAG
# ---------------------------------------------------------

df["requires_review"] = (
    df["duplicate_product_id_flag"] |
    df["invalid_launch_date_flag"] |
    df["negative_revenue_flag"] |
    df["missing_revenue_flag"] |
    df["missing_discount_flag"] |
    df["missing_return_status_flag"] |
    df["revenue_iqr_outlier_flag"]
)

clean_df = df.drop(columns=["launch_date_raw"])

print(clean_df)
Expected cleaned core dataset
product_id	launch_date	product_category	gross_revenue	discount_rate	return_status
PROD_101	2024-01-15	Enterprise Servers	125,400	0.12	No
PROD_102	NaT	Consumer Audio	450	0.00	Missing
PROD_104	2024-03-10	Consumer Audio	-1,250	NaN	Yes
PROD_105	NaT	Enterprise Servers	340,000	0.25	No
PROD_106	2024-04-18	Smart Home	NaN	NaN	No
3. Methodology Justification

The key principle is preservation over invention. Invalid dates are converted to NaT, rather than guessing what somebody intended by February 30 or invalid_date, exactly as prescribed by the supplied rules. Likewise, missing revenue and discounts are left missing instead of being filled with averages that could artificially alter product economics.

The −$1,250 Consumer Audio transaction is deliberately retained. The data dictionary specifically instructs that negative financial values should be flagged as potential refunds or chargebacks rather than automatically dropped. This also aligns with the Return_Status = Yes value on that record, although the snippet alone does not establish that the return caused the negative revenue.

Audit flags provide traceability. An analyst can filter requires_review == True without losing the original business observation.

4. Pattern Discovery & Recommended Visualizations

Three executive-level patterns stand out. First, Enterprise Servers dominate known revenue: their two products account for $465,400 before considering the missing Smart Home revenue, while the two Consumer Audio records net to −$800. This makes a category revenue bar chart the strongest first executive visual.

Second, the apparent Enterprise Servers result is itself concentrated: PROD_105 contributes $340,000, or roughly 73% of all currently known revenue. A product-level ranked bar chart would make that concentration immediately visible.

Third, data completeness limits strategy conclusions. Smart Home has no usable revenue, and two launch dates are unusable. Consequently, category comparisons and launch-date performance analyses should be labeled incomplete rather than interpreted as a complete company performance picture.

For visualization choice, I recommend a horizontal bar chart for gross revenue by product, a category bar chart for aggregate revenue, a boxplot as a diagnostic for revenue dispersion once more observations are available, and a time-series chart only after invalid launch dates are corrected.

Automated EDA — Executive Slate theme
import matplotlib.pyplot as plt
import seaborn as sns

PRIMARY = "#1E293B"
SECONDARY = "#64748B"
GRID = "#E2E8F0"

sns.set_theme(style="whitegrid")
plt.rcParams["axes.edgecolor"] = SECONDARY
plt.rcParams["grid.color"] = GRID

# -------------------------------
# BASIC EDA
# -------------------------------

clean_df.info()

print(
    clean_df.describe(
        include="all",
        datetime_is_numeric=True
    )
)

print("\nMissing values:")
print(clean_df.isna().sum())

print("\nRevenue by category:")
category_revenue = (
    clean_df.groupby("product_category", dropna=False)["gross_revenue"]
            .sum(min_count=1)
            .sort_values(ascending=False)
)
print(category_revenue)


# -------------------------------
# PRODUCT REVENUE
# -------------------------------

plot_df = (
    clean_df.dropna(subset=["gross_revenue"])
            .sort_values("gross_revenue", ascending=True)
)

plt.figure(figsize=(9, 5))
sns.barplot(
    data=plot_df,
    x="gross_revenue",
    y="product_id",
    color=PRIMARY
)

plt.axvline(0, color=SECONDARY, linewidth=1)
plt.title("Apex Dynamics — Gross Revenue by Product")
plt.xlabel("Gross Revenue ($)")
plt.ylabel("Product")
plt.tight_layout()
plt.show()


# -------------------------------
# CATEGORY REVENUE
# -------------------------------

category_plot = (
    clean_df.groupby("product_category", as_index=False)
            .agg(gross_revenue=("gross_revenue", "sum"))
            .sort_values("gross_revenue", ascending=False)
)

plt.figure(figsize=(9, 5))
sns.barplot(
    data=category_plot,
    x="product_category",
    y="gross_revenue",
    color=PRIMARY
)

plt.axhline(0, color=SECONDARY, linewidth=1)
plt.title("Apex Dynamics — Known Revenue by Product Category")
plt.xlabel("Product Category")
plt.ylabel("Gross Revenue ($)")
plt.xticks(rotation=20)
plt.tight_layout()
plt.show()


# -------------------------------
# REVENUE DISTRIBUTION
# -------------------------------

plt.figure(figsize=(8, 4))
sns.boxplot(
    data=clean_df,
    x="gross_revenue",
    color=PRIMARY
)

plt.title("Gross Revenue Distribution")
plt.xlabel("Gross Revenue ($)")
plt.tight_layout()
plt.show()

One caution: with only five unique products and four known revenue observations, the boxplot and statistical anomaly measures are primarily quality-control diagnostics, not strong evidence about the company's broader revenue distribution.

5. Explain to Non-Tech Teams

For an executive audience, the simplest interpretation is: the export wasn't ready for decision-making yet. One product was duplicated because its ID contained hidden spaces, two launch dates could not represent valid dates, several fields used different words to mean “missing,” currency and percentages were stored as formatted text, and one product showed negative revenue.

The cleaning process standardizes all of those conditions without making up information. We remove the duplicate representation of PROD_101, turn monetary and percentage fields into numbers that can actually be calculated, preserve the negative revenue for Finance to investigate, and clearly mark unavailable information instead of guessing.

The preliminary commercial signal is that Enterprise Servers are driving virtually all known positive revenue in this small sample, while Consumer Audio nets negative after the PROD_104 negative transaction. However, Smart Home cannot yet be evaluated because its revenue is missing, and the sample is too small to support broad portfolio conclusions.

Glossary: snake_case means standardized names such as Gross_Revenue → gross_revenue. Vectorization means Pandas processes whole columns efficiently rather than manually handling records one by one. Coercion to NaT means an invalid date becomes Pandas' explicit “Not a Time” marker instead of being guessed. NaN/missing value means the information is unknown, not zero. Imputation means replacing missing information with an estimated value; we deliberately avoid doing that for revenue here. IQR is the middle 50% range of observed values and provides a rule for flagging unusually distant numbers. Outlier means an observation unusually far from the rest—not automatically an error. EDA (Exploratory Data Analysis) is the initial statistical and visual examination used to discover patterns, problems, and questions before formal analysis. pd.to_datetime(..., errors="coerce") converts valid date text into real dates and invalid text into NaT. Audit flag is simply a yes/no marker telling analysts which records deserve additional review.

Where can i put this code to run it?

The easiest options are Google Colab or Jupyter Notebook. Since the code uses Python, Pandas, Matplotlib, and Seaborn, either will work well.

Easiest: Google Colab

Google Colab

You don't need to install Python locally. Open Colab, create a New Notebook, paste the cleaning script into the first code cell, and press the ▶ Run button.

Then paste the visualization/EDA code into a second cell and run that after the cleaning cell. The first cell creates clean_df, which the second cell needs.

Your notebook would essentially look like:

Cell 1
[Cleaning code]
        ↓
Creates clean_df
        ↓
Cell 2
[EDA + visualization code]
        ↓
Charts + statistics
If you already have Python installed

You can also use JupyterLab or Visual Studio Code. For a beginner, though, Google Colab is probably the simplest because there's almost no setup.

One small note: depending on your installed Pandas version, the datetime_is_numeric=True argument in the earlier describe() call may need to be removed.

If you'd like, I can also give you a single copy-and-paste Colab script that cleans the Apex Dynamics data and produces all the charts in one run.

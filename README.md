# Data Quality Audit: Global Superstore

A repeatable, rule-based data quality checklist built with **Python and pandas**. It audits the Global Superstore workbook for **missing, duplicate, range and consistency issues**, quantifies each one, prioritises them, and produces an audit report, an issue log and a cleaned sample.

> **Task 24 | Data Analytics Track**: *Audit a dataset for missing, duplicate, range, and consistency issues. Objective: create a repeatable quality checklist.*

---

## Headline results

| Metric | Result |
|---|---|
| Rows audited | 51,290 order lines, 1,173 returns, 1,590 customers, 13 regional managers |
| Validation rules | **45** across 6 quality dimensions |
| Passed / Failed / Info | **21** / **20** / **4** |
| Failed by priority | **6 P1**, 5 P2, 9 P3 |
| Orders rows with a Medium+ flag | 17,127 (33.4%), excluding the dataset-wide CON-01 issue |

**Verdict:** structurally sound (no nulls in required fields, no exact duplicates, no negative sales, no impossible dates, no orphan keys) but **not safe for time-series, customer or product analysis until the P1 issues are resolved.**

### Top findings

| # | Issue | Scale |
|---|---|---|
| 1 | Order ID year is exactly **5 years earlier** than the Order Date year on every row (CON-01) | 51,290 rows (100%) |
| 2 | **Order IDs reused** by different customers (UNIQ-04) | 659 IDs, 2,701 rows (5.3%) |
| 3 | Every customer name appears **twice** under different IDs with conflicting attributes (UNIQ-07) | 1,590 customers (100%) |
| 4 | One Product ID carries **2 to 4 different product names** (CON-02) | 457 IDs, 4,392 rows (8.6%) |
| 5 | Customer *First Purchase* date is **later than their first order** (CON-10) | 1,072 customers (67.4%) |
| 6 | Returns uses `United States` where Orders uses `US` (CON-09) | 296 rows (25.2%) |

Full write-up: [`reports/Data_Quality_Audit_Report.docx`](reports/Data_Quality_Audit_Report.docx)

---

## Repository structure

```
.
├── data_quality_audit.py        # rule engine + cleaning + output generation
├── requirements.txt
├── data/
│   └── GlobalSuperstore.xlsx    # input (sheets: Orders, Returns, Customer, People)
├── outputs/
│   ├── rule_results.csv         # the checklist: all 45 rules with counts, % and status
│   ├── issue_log.csv            # prioritised issue log with recommended actions
│   ├── row_level_flags.csv      # record-level evidence for every failed rule
│   ├── cleaned_sample.csv       # 1,000-row stratified sample after safe fixes
│   ├── audit_summary.json       # headline metrics
│   └── audit_summary.md         # human-readable summary
└── reports/
    └── Data_Quality_Audit_Report.docx
```

## Quick start

```bash
git clone <your-repo-url>
cd <your-repo>
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python data_quality_audit.py --input data/GlobalSuperstore.xlsx --output outputs
```

Options:

| Flag | Default | Description |
|---|---|---|
| `--input` | `data/GlobalSuperstore.xlsx` | Workbook with sheets `Orders`, `Returns`, `Customer`, `People` |
| `--output` | `outputs` | Folder for generated files |
| `--sample-size` | `1000` | Rows in the cleaned sample (stratified by Market) |
| `--seed` | `42` | Random seed so the sample is reproducible |

---

## The quality checklist

Every rule is one `Rule(...)` entry with a function that returns the records that **fail**. Rules cover six dimensions:

| Dimension | Example rules | Pass / Total |
|---|---|---|
| **Completeness** | Required fields not null; US rows have a Postal Code | 3 / 4 |
| **Uniqueness** | Row ID unique; no duplicate rows; one customer per Order ID | 3 / 7 |
| **Validity** | ID formats (regex), allowed values, whitespace, data types | 3 / 9 |
| **Range** | Sales > 0; Discount in [0, 1]; Ship Date >= Order Date; loss <= revenue | 7 / 10 |
| **Consistency** | One name per Product ID; one market per country; cross-table agreement | 2 / 11 |
| **Integrity** | Orders.Customer ID exists in Customer; Returns.Order ID exists in Orders | 3 / 4 |

### Adding a rule

```python
add(Rule("RNG-11", "Orders", "Range", "Quantity at most 100",
         "Quantity above 100 is implausible for this catalogue.", 3,
         lambda T: _o(T)["Quantity"] > 100,      # returns a boolean mask of FAILING rows
         column="Quantity",
         action="Check source system."))
```

Re-run the script and the new rule appears in `rule_results.csv`, `issue_log.csv` and the row-level flags.

## How issues are quantified and prioritised

Each failed rule reports **records failed** and **% of records** in its table. Priority is calculated, not guessed:

```
Priority score = Impact (1-5) x Prevalence band (1-5)
```

| Prevalence band | % of records affected |
|---|---|
| 1 | under 0.1% |
| 2 | 0.1% to under 1% |
| 3 | 1% to under 5% |
| 4 | 5% to under 20% |
| 5 | 20% or more |

| Priority | Score | Meaning |
|---|---|---|
| **P1** | 15 or more | Fix before analysis |
| **P2** | 8 to 14 | Fix this cycle |
| **P3** | below 8 | Fix when convenient |
| **Info** | n/a | Expected or review-only (e.g. Postal Code not captured outside the US) |

**Impact scale:** 5 = data cannot be trusted (e.g. broken key), 4 = a core metric is wrong, 3 = a segment of analysis is wrong, 2 = cosmetic, 1 = informational.

## Cleaning policy: fix what is deterministic, flag the rest

Only the Orders table is cleaned. Safe, reversible fixes are applied; anything needing a business decision is left in place and **flagged** in the `dq_flags` column.

| Rule | Fix | Failures before | After |
|---|---|---|---|
| VAL-05 | Trim whitespace in text columns | 16 | 0 |
| VAL-03 | Remove embedded space in Product ID (`TEC-HP -10003345`) | 83 | 0 |
| VAL-06 | Round Discount to 3 decimals (`0.15000000000000002` to `0.15`) | 821 | 0 |
| VAL-07 | Store Postal Code as text; zero-pad US ZIP codes to 5 characters | 449 | 0 |
| CON-09 | Map Returns Market `United States` to `US`; de-duplicate Returns; derive `Returned` (Yes/No) | 296 | 0 label mismatches |

**Deliberately not auto-fixed** (needs a data-owner decision): the Order ID / Order Date year offset, reused Order IDs, duplicate customers, conflicting product names, EMEA vs AMEA region label, and the First Purchase date conflict.

## Outputs explained

| File | Columns / content |
|---|---|
| `rule_results.csv` | `rule_id, table, dimension, rule, description, column, records_checked, records_failed, pct_failed, status, severity, impact, prevalence_band, priority_score, priority, fix_policy, recommended_action` |
| `issue_log.csv` | `issue_id, rule_id, priority, priority_score, severity, table, dimension, rule, column, records_failed, records_checked, pct_failed, fix_policy, recommended_action, status_workflow` |
| `row_level_flags.csv` | `rule_id, table, record_key, column, value, severity`. Trace any failure back to the source row (Orders by `Row ID`, Customer by `Customer ID`, Returns by `Order ID`) |
| `cleaned_sample.csv` | 1,000 Orders rows after fixes, plus `Returned` and `dq_flags` columns |

> `row_level_flags.csv` is about 4.7 MB. Add it to `.gitignore` if you prefer a lighter repository; it is regenerated on every run.

---

## Interview notes

**What makes a quality rule useful?**
It is **specific, measurable, tied to a business consequence, actionable and repeatable**. It states exactly what must be true ("a Product ID maps to one Product Name"), returns a count and percentage rather than "looks wrong", and names who must act. It earns its place only if a failure would change a decision, which is why every rule here carries an impact score. It should run automatically on every load, leave a record-level trail, and avoid noise: by-design gaps and rules that flag every row are marked Info so real problems stay visible.

**How do you prioritise issues?**
By **impact x prevalence**, then adjusted for effort. Impact asks what breaks downstream; prevalence asks how many records are affected. A cheap fix with wide reach (the Returns Market label) is a quick win; a high-impact issue that needs a business decision (which date field is correct) is escalated immediately even though it cannot be auto-fixed.

## Limitations

- The audit finds *internal* inconsistencies. It cannot prove a value is true against the real world.
- Rules use thresholds chosen for this dataset (for example 3 x IQR for outliers, minimum customer age of 18); adjust them in `data_quality_audit.py`.
- Fixes are applied to a sample for demonstration. Apply them upstream or in your ETL before using the full table.

## Tech stack

Python 3.10+, pandas, NumPy, openpyxl, tabulate.

## License

Add your preferred license (for example MIT) before publishing. The Global Superstore dataset is a public sample dataset; check its original terms before redistributing the file.

# 🧹 Data Cleaning Pipeline

> **Run this first before any other notebook in this suite.**

Transforms raw Wise Waste MIS exports into clean, analysis-ready CSVs — handling inconsistent column names, merged header rows, cost duplication across multi-material records, and missing derived fields automatically.

---

## The Problem It Solves

Raw MIS exports from Wise Waste arrive differently across facilities and over time:
- Column names change (`"Dispatched Qty (in KG)"` vs `"Dispatched Quantity"` vs `"dispatched qty"`)
- Multi-material rows repeat trip-level costs (transport, loading) across every material row — causing double-counting if summed directly
- Forward-fill is needed for header rows but must not touch per-material fields like quantity and rate
- Derived columns like `Net Procurement Cost` and `Net Material Sales Cost` are either missing or need to be recomputed from scratch

Doing this manually for each export takes **45–90 minutes per file**. This pipeline does it in **under 30 seconds**.

---

## Architecture

```
Raw CSV Upload
      │
      ▼
┌─────────────────────────────┐
│  1. Dataset Auto-Detection  │  ← filename keywords + column header matching
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  2. Column Aliasing         │  ← 100+ variant names → canonical schema
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  3. Duplicate Col Coalesce  │  ← merge split columns by combine_first()
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  4. Selective Forward-Fill  │  ← fills header rows, skips material rows
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  5. Cost Splitting          │  ← transport/loading split equally per code
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  6. Derived Column Compute  │  ← Accepted Qty, Net Procurement Cost, etc.
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  7. Column Selection        │  ← keep only required output columns
└────────────┬────────────────┘
             │
             ▼
   cleaned_datasets.zip
```

---

## Languages & Libraries

| Layer | Tool | Purpose |
|---|---|---|
| Language | Python 3 | Core logic |
| Data processing | Pandas | DataFrame operations, ffill, groupby |
| Numeric ops | NumPy | Safe numeric coercion |
| Text matching | `re` (regex) | Column name canonicalization |
| File handling | `io`, `os`, `zipfile` | CSV parsing, ZIP output |
| Environment | Google Colab | `files.upload()` / `files.download()` |

---

## Features in Detail

### Auto-Detection (3-pass)
1. Filename keyword match (`inward`, `production`, `outward`)
2. Exact column header match against known detect columns
3. Canonicalized column header match (lowercase, strip punctuation)

### Column Aliasing
Maps **100+ raw column name variants** to a canonical schema. Examples:
- `"Transportation Cost in Rs"` → `"Transportation Cost"`
- `"Dispatched Qty (in KG)"` → `"Dispatched Quantity"`
- `"No. of Present Staffs"` → `"No. of Staff Present"`
- `"Gsp Photo 1"` → `"GPS Photo 1"`

### Selective Forward-Fill
Trip-level fields (vendor, date, vehicle, driver) are forward-filled across multi-material rows. Per-material fields (quantity, rate, rejection) are **never** forward-filled.

### Cost Splitting
Shared trip costs are split equally across all material rows belonging to the same Inward/Outward Code. Without this, summing transport cost across a 3-material trip would triple-count it.

### Derived Columns Computed

**Inward:**
- `Accepted Quantity` = Received Quantity − Rejected Quantity
- `Value of Received Material` = Received Quantity × Rate
- `Value of Accepted Material` = Accepted Quantity × Rate
- `Net Procurement Cost` = Value of Accepted Material − Transport − Additional − Loading

**Outward:**
- `Accepted Quantity` = Dispatched Quantity − Rejected Quantity
- `Value of Dispatched Material` = Dispatched Quantity × Rate
- `Value of Accepted Material` = Accepted Quantity × Rate
- `Total Incentive Cost` = Incentive per Kg × Accepted Quantity
- `Net Material Sales Cost` = Value of Accepted Material + Incentive − Transport − Additional − Loading

---

## Supported Datasets

| Dataset | Detection | Output Columns |
|---|---|---|
| Inward | `inward` in filename or `Inward Code` column | 27 standardized columns |
| Outward | `outward` in filename or `Outward Code` column | 32 standardized columns |
| Production | `production` in filename or `Production Code` column | 12 standardized columns |

---

## Use Cases

- **Before every analytics run** — clean one or multiple raw exports in one step
- **Cross-facility standardization** — different facilities export slightly different column names; this normalizes all of them
- **New staff onboarding** — no manual column renaming or formula work needed
- **Data audit prep** — clean output can be shared with auditors or ULB stakeholders directly

---

## Time Saved

| Task | Manual (Excel) | This Pipeline |
|---|---|---|
| Column renaming across 1 file | 15–20 min | < 5 sec |
| Cost splitting across multi-material rows | 20–30 min | < 5 sec |
| Derived column formulas (Net Procurement Cost etc.) | 15–20 min | < 5 sec |
| Processing 3 files at once | 90–150 min | ~15 sec |
| **Total per export cycle** | **~2 hours** | **< 30 sec** |

**Estimated time saving: ~98% reduction per cleaning cycle.**

---

## Input / Output

| | Detail |
|---|---|
| **Input** | Raw Inward / Outward / Production CSV from Wise Waste MIS |
| **Output** | `<filename>_cleaned.csv` per file + `cleaned_datasets.zip` auto-download |

---

## How to Use

### Google Colab
1. Click **Open in Colab** above
2. Run the single cell — you'll be prompted to upload files
3. Upload one or more raw CSVs (any combination of Inward, Outward, Production)
4. Each file is auto-detected, cleaned, and bundled — `cleaned_datasets.zip` downloads automatically


```

---

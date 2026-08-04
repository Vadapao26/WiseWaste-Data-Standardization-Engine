🧹 WiseWaste Data Engine

Run this first before syncing to Supabase, and before any analytics.

Transforms raw Wise Waste MIS exports into clean, enriched, analysis-ready files — in one pass. Handles the messy reality of real-world exports (inconsistent column names, merged header rows, shared costs duplicated across multi-material rows, missing derived fields) and adds the Source Type / Vendor Type / Authorisation / Material Category fields the analytics layer depends on, by matching each row against the facility's vendor and material mapping tables.

This replaces the two-notebook workflow (Data_Cleaning.ipynb here, plus a separate enrichment notebook) with a single script. Data_Cleaning.ipynb still works and is left in this repo unchanged — wisewaste_data_engine.py is the merged, recommended path going forward.

The Problem It Solves

Raw MIS exports from Wise Waste arrive differently across facilities and over time, and the analytics layer needs more than just clean columns:

Column names change ("Dispatched Qty (in KG)" vs "Dispatched Quantity" vs "dispatched qty")
Multi-material rows repeat trip-level costs (transport, loading) across every material row — causing double-counting if summed directly
Forward-fill is needed for header rows but must not touch per-material fields like quantity and rate
Derived columns like Net Procurement Cost and Net Material Sales Cost are either missing or need to be recomputed from scratch
Every vendor+location and customer+destination pair needs a Source Type / Vendor Type and Authorisation looked up by hand, against a mapping sheet with 150+ entries
Every material needs a category assigned from an 80+ entry lookup

Doing this manually for each export takes 60–120 minutes per file, mapping lookups included. This pipeline does it in under 30 seconds.

Architecture
Raw CSV / Multi-Sheet Excel Workbook Upload
     (any subset — one file works on its own)
      │
      ▼
┌──────────────────────────────┐
│  1. Dataset Auto-Detection   │  ← filename keywords + column header matching
│                               │     (Inward/Outward/Production, and now also
│                               │      Expense/Revenue/Training)
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  2. Column Aliasing          │  ← 100+ variant names → canonical schema
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  3. Duplicate Col Coalesce   │  ← merge split columns by combine_first()
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  4. Selective Forward-Fill   │  ← fills header rows, skips material rows
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  5. Cost Splitting           │  ← transport/loading split equally per code
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  6. Derived Column Compute   │  ← Accepted Qty, Net Procurement Cost, etc.
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  7. Vendor / Customer         │  ← Source Type, Vendor Type, Authorisation
│     Mapping Enrichment        │     (155 vendor+location pairs,
│     (Inward + Outward only)   │      80 customer+destination pairs)
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  8. Material Category         │  ← 83 material → category mappings
│     Mapping                   │
└────────────┬──────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  9. Column Selection          │  ← keep only required output columns
└────────────┬──────────────────┘
             │
             ▼
   <name>_cleaned.csv  /  <name>_enriched.xlsx

Production, Expense, Revenue, and Training pass through steps 1–6 (Production) or are left untouched apart from date formatting (Expense/Revenue/Training) — enrichment (steps 7–8) only applies to Inward and Outward, since Source Type and Material Category are vendor/material concepts that don't apply to the other datasets.

Languages & Libraries
Layer	Tool	Purpose
Language	Python 3	Core logic
Data processing	Pandas	DataFrame operations, ffill, groupby
Numeric ops	NumPy	Safe numeric coercion
Text matching	re (regex)	Column name canonicalization
Excel I/O	openpyxl	Multi-sheet workbook read/write
File handling	io, os, zipfile	CSV/Excel parsing, ZIP output
Environment	Google Colab or local Python	files.upload()/files.download() in Colab; CLI args locally
Features in Detail
Auto-Detection (3-pass, now covers 6 dataset types)
Filename keyword match (inward, production, outward, expense, revenue, training)
Exact column header match against known detect columns
Canonicalized column header match (lowercase, strip punctuation)
Column Aliasing

Maps 100+ raw column name variants to a canonical schema. Examples:

"Transportation Cost in Rs" → "Transportation Cost"
"Dispatched Qty (in KG)" → "Dispatched Quantity"
"No. of Present Staffs" → "No. of Staff Present"
"Gsp Photo 1" → "GPS Photo 1"
Selective Forward-Fill

Trip-level fields (vendor, date, vehicle, driver) are forward-filled across multi-material rows. Per-material fields (quantity, rate, rejection) are never forward-filled.

Cost Splitting

Shared trip costs are split equally across all material rows belonging to the same Inward/Outward Code. Without this, summing transport cost across a 3-material trip would triple-count it.

Derived Columns Computed

Inward:

Accepted Quantity = Received Quantity − Rejected Quantity
Value of Received Material = Received Quantity × Rate
Value of Accepted Material = Accepted Quantity × Rate
Net Procurement Cost = Value of Accepted Material + Transport + Additional + Loading

Outward:

Accepted Quantity = Dispatched Quantity − Rejected Quantity
Value of Dispatched Material = Dispatched Quantity × Rate
Value of Accepted Material = Accepted Quantity × Rate
Total Incentive Cost = Incentive per Kg × Accepted Quantity
Net Material Sales Cost = Value of Accepted Material + Incentive − Transport − Additional − Loading
Vendor / Customer Mapping Enrichment (new)

Every Inward row's Received Material From + Vendor Location pair, and every Outward row's Customer + Destination pair, is matched against a lookup table (exact match first, then case-insensitive, then vendor-name-only fallback) to assign:

Inward: Source Type (ULB / DWCC / SHG / Waste Picker / Aggregator / Bulk Waste Generator) + Authorisation (Registered/Unregistered)
Outward: Vendor Type (Recycler / Aggregator / Co-Processing / Returned to Source) + Authorisation

Unmatched pairs get "Unknown" rather than a guess, and are printed as a coverage report after each run — so a new vendor showing up in a sheet is visible immediately instead of silently falling through.

Material Category Mapping (new)

Every Material value is matched against an 83-entry table (exact match, then case-insensitive, then substring fallback) to assign Inward Material Category / Outward Material Category (Rigid Plastics, Flexible Plastics, Metals, Paper, PS, Glass, TetraPak, Co-Processing, Rejects, E-waste, Others).

Flexible Input — No Restrictions (new)
One standalone CSV (just Inward.csv) works on its own — no other datasets required
Any combination of standalone CSVs in one batch
One combined multi-sheet Excel workbook, with any subset of the six sheets present
A mix of standalone CSVs and workbooks in the same batch
An unrecognized file is skipped with a warning, not a crash
Supported Datasets
Dataset	Detection	Cleaning	Enrichment	Output Columns
Inward	inward in filename or Inward Code column	✅	✅ Source Type, Authorisation, Material Category	29 standardized columns
Outward	outward in filename or Outward Code column	✅	✅ Vendor Type, Authorisation, Material Category	36 standardized columns
Production	production in filename or Production Code column	✅	—	12 standardized columns
Expense	expense in filename or Expense Code column	dates only	—	unchanged
Revenue	revenue in filename or Revenue Code column	dates only	—	unchanged
Training	training in filename or Training Code column	dates only	—	unchanged
Use Cases
Before every sync to Supabase — this is the step that fills in Source Type, Vendor Type, Authorisation, and Material Category; sync_to_supabase.py only casts and renames columns, it never derives these values itself
Before every analytics run — clean and enrich one or multiple raw exports in one step
Cross-facility standardization — different facilities export slightly different column names; this normalizes all of them
New staff onboarding — no manual column renaming, formula work, or mapping-sheet lookups needed
Data audit prep — clean, enriched output can be shared with auditors or ULB stakeholders directly
Finding unmapped vendors — the coverage report after each run flags any vendor/customer pair not yet in the mapping tables
Time Saved
Task	Manual (Excel)	This Pipeline
Column renaming across 1 file	15–20 min	< 5 sec
Cost splitting across multi-material rows	20–30 min	< 5 sec
Derived column formulas (Net Procurement Cost etc.)	15–20 min	< 5 sec
Vendor/customer mapping lookups (150+ entries)	20–30 min	< 5 sec
Material category lookups (80+ entries)	10–15 min	< 5 sec
Processing 3 files at once	100–170 min	~15 sec
Total per export cycle	~2.5 hours	< 30 sec

Estimated time saving: ~99% reduction per cleaning-and-enrichment cycle.

Input / Output
	Detail
Input	Raw Inward / Outward / Production / Expense / Revenue / Training CSV, or a combined multi-sheet Excel workbook — any subset, in any combination
Output	<filename>_cleaned.csv per standalone dataset, or <filename>_enriched.xlsx per workbook (all sheets preserved)
How to Use
Google Colab
Open wisewaste_data_engine.py in Colab (or paste into a new notebook cell)
Run the cell — you'll be prompted to upload files
Upload any combination: one file, several standalone CSVs, or a combined workbook
Cleaned/enriched output downloads automatically (cleaned_datasets.zip for multiple standalone files, or the enriched workbook directly)
Local
bash
python wisewaste_data_engine.py Inward.csv
python wisewaste_data_engine.py Inward.csv Outward.csv Production.csv
python wisewaste_data_engine.py Facility_Raw_Data.xlsx
Updating the Mapping Tables

When Mapping_Sheet.xlsx changes (new vendors, new material categories), update MATERIAL_CATEGORY_MAP, VENDOR_SOURCE_MAP, or CUSTOMER_DESTINATION_MAP directly in wisewaste_data_engine.py — they're plain Python dicts, no build step. pull_all_unique_pairs.py (in the waste-ops-mis repo) is the tool for finding which vendor/customer pairs are still missing a mapping.

Validation

Before being added to this repo, wisewaste_data_engine.py was:

Run against real Inward.csv, Outward.csv, Production.csv, Expense.csv, Revenue.csv exports, both standalone and combined into a test workbook
Diffed column-for-column against a known-correct enriched reference file — exact match on Inward, Outward, and Production
Diffed byte-for-byte against the original mapping tables — zero data loss in the merge
Confirmed 100% mapping coverage (no "Unknown" rows) against real Inward/Outward data

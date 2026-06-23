1. 🧹 Data Cleaning Pipeline

data-cleaning-pipeline/

Run this first, before any other notebook.

Takes raw operational CSV exports from the Wise Waste MIS and standardizes them into clean, analysis-ready files. Handles the messy reality of real-world exports — inconsistent column names across facilities, merged header rows, shared costs duplicated across multi-material rows, and missing derived fields.

What it does:


Auto-detects dataset type from filename or column headers — Inward, Production, or Outward
Column aliasing — maps 100+ raw column name variants to a canonical schema (e.g. "Dispatched Qty (in KG)" → "Dispatched Quantity")
Duplicate column coalescing — merges split columns by combining non-null values
Selective forward-fill — fills grouped header rows while leaving per-material rows (quantity, rate, rejection) untouched
Cost splitting — splits shared costs (transport, loading, additional) equally across all material rows per code, preventing double-counting
Derived column computation — calculates Accepted Quantity, Value of Received/Dispatched Material, Net Procurement Cost, Net Material Sales Cost, Total Incentive Cost
Batch processing — upload multiple CSVs at once; each detected, cleaned, and bundled into a single ZIP download


Supported datasets:

DatasetDetected byKey derived columnsInwardinward in filename or Inward Code columnAccepted Quantity, Value of Accepted Material, Net Procurement CostOutwardoutward in filename or Outward Code columnAccepted Quantity, Value of Accepted Material, Net Material Sales Cost, Total Incentive CostProductionproduction in filename or Production Code columnStandardization + forward-fill only

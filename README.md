Component Availability Control Tower — Technical Handover

1. Overview
The Component Availability Control Tower is a single self-contained HTML file that turns the FG/Component Stock vs Demand SQL report (the SEL+SAL / SC01-variant cascading BOM-netting query) into an interactive planning tool. It is built for supply planners and procurement to answer three questions the raw report output can't answer on its own: which finished goods (FGs) are blocked and why, which components to chase first for the most completion impact, and what happens if a given quantity of a component arrives.
Status: v1, shipped. Fully offline — no CDN, no external fonts, no network calls of any kind. The person opens it by double-clicking the file; there is no server, no build step, no install.
Who uses it: supply/production planners (FG Focus, Supply Simulator), procurement (Procurement Priority, Stock Visibility), and the ERP/Digital Transformation owner (Data Integrity tab, for validating the query's own output before it's trusted).
Where the source query lives: the tool is presentation-only. It has no knowledge of SAP or the SQL itself — it only consumes whatever the query's SELECT statement outputs (currently the v1.4 DRAFT SC01-hardcoded-variant query, 24 columns, one row per BOM-cascade demand line). Any change to the query's output columns needs a matching change to this tool's EXPECTED/REQUIRED column lists (see Section 3).
2. Architecture
One HTML file, ~530 KB, three parts in order:
1.	<head> — inline <style> (all CSS, no external stylesheet) and one inline <script> containing the embedded parser library, xlsx.core.min.js from SheetJS (npm xlsx@0.18.5), pasted in verbatim. This is what makes the file work with zero internet access — the Excel-parsing engine ships inside the HTML instead of being fetched from a CDN.
2.	<body> — two top-level screens toggled by display:none/block: #landing (the upload/dropzone screen) and #app (everything after a file loads). #app contains the plant-filter bar, KPI row, narrative card, the tab nav, and nine <section class="panel"> blocks, one per tab.
3.	Closing <script> — all application logic, wrapped in one IIFE, vanilla JS (no framework, no build step — what you see in the file is exactly what runs).
Why xlsx.core.min.js and not xlsx.full.min.js: the full build embeds legacy codepage conversion tables that contain literal U+FFFD replacement characters (by design, for byte values with no character mapping). That's harmless for parsing but was rejected when re-publishing this file as a hosted Claude artifact. The core build (which is about half the size) has none of those characters and still handles .xlsx/.csv/.xls reading, date coercion (cellDates:true), and sheet_to_json — everything this tool uses. If a future change needs a xlsx.full.min.js-only feature (e.g. certain legacy codepages), swap the embedded library back in (see Section 7).
No framework, no dependencies beyond the embedded library. No React, no build tooling, no package.json shipped in the file. This keeps it a single artifact that survives being emailed, copied to a USB drive, or opened on a locked-down plant floor machine with no internet.
State model: everything lives in module-level JS variables inside the IIFE — ROWS (every parsed row), LEVEL1_ROWS (a filtered view, bom_level===1), FG_AGG (precomputed per-FG summary), STOCK_INDEX (derived stock positions), SHORT_ROWS_BY_COMP (index for the simulator), and CURRENT_PLANT (the active plant filter). There's no framework-level reactivity — every render function reads directly from these globals and rewrites the relevant DOM subtree with innerHTML. refreshAll() is the one function that re-runs every render function in sequence; it's called once after file parse and again every time the plant filter changes.

3. Data Model
Input: any .xlsx/.xls/.csv whose header row (matched case-insensitively, order-independent) contains at minimum the columns in REQUIRED, ideally all 24 in EXPECTED. If the sheet is a workbook with multiple tabs, the tool looks for one named exactly Query result, else falls back to the first sheet.
Column	Used for
plant, company_code	plant filter, scope detection
fg_material, fg_description	FG Focus search/list
bom_level	Level 1 = FG's direct components; Level 2+ = upstream sub-assembly cascade
parent_material, bom_item_number, bom_item_path	cascade structure, priority-order tiebreak
component_material, component_description, component_uom	every component-level view
sales_order, item_number, schedule_line	demand identity, priority order
requirement_date	aging buckets, simulator priority order
fg_demand, bom_qty_per_fg	FG open-quantity rollup
gross_child_part_demand_to_cover_fg	this line's own gross demand (not cumulative)
gross_child_part_stock	running stock position after prior lines — see the stock-derivation note below
gross_shortage_or_excess, gross_status_flag	legacy gross view (pre-netting)
net_demand_this_line, net_coverage_flag	the netted shortfall this tool is built around
bom_number	not currently used in the UI
Parsed row object (ROWS[i]): every source column, type-coerced (numbers via Number(), dates via cellDates → stripped to midnight local time), plus five derived fields computed once at load:
•	_covBucket — 'FULLY' | 'PARTIALLY' | 'NOT COVERED' | 'UNKNOWN', parsed from the emoji-prefixed net_coverage_flag text by substring match (indexOf('NOT COVERED') etc.) — not an exact-match lookup, so it tolerates minor wording changes in the SQL's CASE statement.
•	_monthDiff — signed integer, (current_year*12+current_month) − (req_year*12+req_month). Positive = requirement month is in the past. This is what Section 6 explains in detail; it replaced an earlier day-level calculation.
•	_key — composite string (plant|component|SO|item|schedule|bom_item_path) used only by the Data Integrity tab's duplicate-row check.
•	_sortKey — composite string matching the SQL's own ORDER BY (requirement_date → sales_order → item_number → schedule_line → bom_item_path), used by the Supply Simulator to replay the query's priority queue exactly.
LEVEL1_ROWS is simply ROWS.filter(bom_level===1) — precomputed once because it's the source for FG Focus and the "direct" scope of Procurement Priority, both of which re-render often.

---
name: analyze-opportunity
description: Investigates a land parcel and its adjacent neighboring parcels for real estate investment opportunity analysis, including exact acquisition cost-per-acre valuation benchmarking and neighboring flood/hydrological risk investigation.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Opportunity Skill

When asked to investigate a property or analyze an investment opportunity for a lot:
1. **Retrieve Subject Lot Data**: Use the `get_lot_data` tool to lookup the subject property attributes by its lot ID (`plot_id`).
2. **Retrieve All Adjacent/Neighboring Parcel Data (MANDATORY)**: Inspect the `adjacent_parcels` list returned in the subject lot's data. Immediately call `get_lot_data` for **every single lot ID** listed in `adjacent_parcels` so you have complete FEMA flood zone, topography, soil classification, easement, acreage, and sale price data for all neighboring properties.
3. **Derive and Benchmark Acquisition Cost-Per-Acre (Core Valuation Metric)**:
   - Explicitly calculate and state the exact mathematical **Acquisition Cost-Per-Acre** using the formula:
     `Acquisition Cost-Per-Acre = Last Sale Price / Acreage`
   - Show the exact formula, numbers, and resulting dollar amount per acre (formatted to 2 decimal places, e.g., `$450,000 / 1.2 acres = $375,000.00 per acre`).
   - Highlight **Acquisition Cost-Per-Acre** prominently as a **core valuation metric** in both the `## Executive Summary` and `## Acquisition Basis and Holding Costs` sections (including in a Markdown valuation summary table).
   - Explicitly benchmark the subject lot's Acquisition Cost-Per-Acre against:
     (a) the subject lot's **Assessed Land Value Per Acre** (`Assessed Land Value / Acreage` from tax records), and
     (b) the **Acquisition Cost-Per-Acre of each adjacent/neighboring parcel** (`neighbor last_sale_price / neighbor acreage`), analyzing whether the subject parcel is priced at a premium or discount per acre relative to adjacent land and zoning density.
4. **Investigate Flood & Neighboring Hydrological Risks**: Consult the `analyze-flood-risk` skill to thoroughly evaluate both the subject parcel's exact FEMA flood zone and the exact FEMA flood zones, topography, soils, and shared drainage/runoff impacts of all adjacent/neighboring parcels.
5. **Investigate Tax & Carrying Costs**: Consult the `analyze-tax-implications` skill to retrieve tax assessment data via `get_tax_assessment_data` using `parcel_apn`, and compute exact monthly tax holding costs (`annual_tax_amount / 12`).
6. **Synthesize Comprehensive Findings**: Present clear Markdown summary tables, explicit mathematical derivations, utility/topography CapEx analysis, Phase I/II ESA due diligence steps, and exhaustive neighboring flood risk analysis across all sections of the memorandum.

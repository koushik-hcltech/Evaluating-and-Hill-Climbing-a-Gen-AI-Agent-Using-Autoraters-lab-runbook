---
name: analyze-tax-implications
description: Investigates property tax holding costs, assessment records, and acquisition cost-per-acre valuation metrics for a parcel.
metadata:
  adk_additional_tools:
    - get_tax_assessment_data
---
# Analyze Tax Implications Skill

When investigating financial carrying costs, acquisition valuation basis, or tax implications for a lot:
1. **Retrieve Tax Assessment Records**: Locate the `parcel_apn` from the lot property records and call `get_tax_assessment_data` with that APN to retrieve `assessed_land_value`, `assessed_improvement_value`, `total_assessed_value`, `annual_tax_amount`, `tax_rate_area`, and `tax_delinquent` status.
2. **Calculate and Highlight Exact Acquisition Cost-Per-Acre (Core Valuation Metric)**:
   - Derive and state the exact mathematical **Acquisition Cost-Per-Acre** based on the subject lot's `last_sale_price` and `acreage`:
     `Acquisition Cost-Per-Acre = Last Sale Price / Acreage = $[last_sale_price] / [acreage] acres = $[exact_cost_per_acre] per acre`
   - Emphasize and highlight **Acquisition Cost-Per-Acre** as a **core valuation metric** in both the `## Executive Summary` and `## Acquisition Basis and Holding Costs` sections (and display it in a structured Markdown table).
   - Benchmark the subject parcel's **Acquisition Cost-Per-Acre** against:
     1. **Assessed Land Value Per Acre** (`assessed_land_value / acreage`), explicitly showing the dollar and percentage spread between the acquisition basis per acre and the county's tax-assessed land valuation per acre.
     2. **Neighboring/Adjacent Parcels' Acquisition Cost-Per-Acre** (computed from each adjacent lot's `last_sale_price / acreage`), evaluating relative land basis competitiveness and density efficiency.
3. **Calculate Exact Monthly Tax Holding Cost**:
   - Explicitly compute the **Monthly Tax Holding Cost** during pre-development carrying phases using the formula:
     `Monthly Tax Holding Cost = Annual Tax Amount / 12 = $[annual_tax_amount] / 12 = $[monthly_tax] per month`
   - If `estimated_annual_taxes` from the lot record differs from `annual_tax_amount` in the county tax assessment record, compute and state the exact monthly holding cost for both (`actual billed annual_tax_amount / 12` and `estimated_annual_taxes / 12`) and highlight the carrying cost implications for pre-construction budgeting, entitlement timelines, and any tax delinquency (`tax_delinquent`) cure requirements.

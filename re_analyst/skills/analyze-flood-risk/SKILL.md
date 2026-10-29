---
name: analyze-flood-risk
description: Investigates FEMA flood zone designations, neighboring/adjacent property flood zones, comparative hydrological/drainage/runoff risks, and environmental risks for real estate investment opportunity analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Flood Risk Skill

When evaluating environmental, flood, or hydrological risks for a subject lot:
1. **Subject Parcel FEMA Flood Zone & Construction/Insurance Impacts**:
   - Explicitly report the exact `flood_zone` designation of the subject parcel (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone X (Minimal Risk)`).
   - Detail specific vertical construction and civil engineering requirements (e.g., Base Flood Elevation (BFE) compliance, structural fill dirt importation and compaction for Zone AE, deep driven pilings/columns and breakaway walls for Zone VE coastal wave velocity action, elevated mechanical/electrical pads, compensatory flood storage, and FEMA NFIP mandatory flood insurance premium carrying impacts).
2. **Neighboring / Adjacent Property Flood Risk Investigation (MANDATORY)**:
   - Check the `adjacent_parcels` field of the subject lot.
   - For **every adjacent lot ID** listed in `adjacent_parcels`, call `get_lot_data` (if not already retrieved) to inspect the neighboring parcel's exact `flood_zone`, `topography`, `soil_classification`, `acreage`, `zoning`, and `easement_type`.
3. **Exhaustive Comparative Hydrological, Drainage & Runoff Analysis**:
   - Include a dedicated subsection and Markdown comparison table under `## Risk Assessment` (and summarize in `## Executive Summary`) comparing the subject lot with each adjacent parcel by exact Lot ID.
   - Explicitly state the exact FEMA `flood_zone` classification, `topography` grade/slope, `soil_classification`, and `easement_type` for every neighboring property.
   - Critically analyze how neighboring properties impact the subject parcel's hydrology and development feasibility:
     - **Cross-Boundary Stormwater Runoff & Topographic Gradients**: Analyze how slope differentials between neighboring lots (e.g., steep `15%+ grade`, moderate `5-12% grade`, gentle `2-5% grade`, or flat `0-2% grade`) and the subject parcel create sheet flow, surface water runoff vectors, or ponding/backwater accumulation onto the subject parcel.
     - **Floodplain Contiguity & Hydraulic Connectivity**: Evaluate whether neighboring parcels share a high-risk floodplain (`Zone AE` or `Zone VE`) or form a transition boundary with `Zone X`, and explain how floodwater staging, coastal wave propagation, or displaced runoff from adjacent development affects the subject site.
     - **Soil Permeability & Easement Interactions**: Analyze how the soil types of the subject and adjacent parcels (e.g., `Clay loam (Moderate shrink-swell potential)` impeding infiltration and increasing surface runoff vs. `Sandy loam (Good drainage)` vs. `Urban Land-Inwood complex` imperviousness) and any existing `Drainage Easement` or utility/access easements govern cross-lot drainage capacity, required stormwater detention/retention vaults, perimeter swales, interceptor trenches, and civil engineering CapEx.
4. **Environmental Status (Phase I / Phase II ESA)**:
   - Explicitly report the exact `environmental_status` (e.g., `Phase I ESA - Clean`, `Phase I ESA - Recognized Environmental Condition (REC flagged)`, or `Phase II ESA Required (Historical Underground Storage Tank)`) and prescribe actionable due diligence and remediation steps (e.g., Phase II subsurface soil/groundwater vapor intrusion testing, UST removal, environmental escrow holdbacks).

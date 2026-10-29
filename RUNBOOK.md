# Evaluating and Hill Climbing a Gen AI Agent Using Autoraters — Complete Lab Runbook & Codebase

## Overview
In this challenge lab, you evaluate and iteratively improve ("hill climb") a Generative AI Real Estate Investment Analyst agent (`re_analyst`) built with the **Google Agent Development Kit (ADK)** using **LLM-as-a-Judge autoraters** powered by the **Vertex AI Gen AI Evaluation Service (GEAP)**.

### Lab Objectives
1. **Generate Agent Responses**: Execute the `re_analyst` ADK agent across a batch of target property IDs to generate a baseline dataset of investment opportunity reports (`EvalSet`).
2. **Evaluate with LLM-as-a-Judge**: Use the Gen AI Evaluation Service on Vertex AI (`vertexai.evaluation.EvalTask`, `PointwiseMetric`, `PointwiseMetricPromptTemplate`) to score the agent's reports against a 19-criterion rubric (`eval_harness/rubrics.json`).
3. **Analyze Evaluation Self-Consistency**: Run multi-pass evaluations ($k$-passes) and analyze intra-rater reliability metrics (**Krippendorff's Alpha**, **Pairwise Percent Agreement**, **Max Score Range**, and **Standard Deviation of Report Means**) to distinguish reliable rubric criteria from noisy criteria and **homogeneity traps**.
4. **Hill Climb Agent Quality**: Diagnose agent deficiencies using autorater rationales and iteratively refine the agent's ADK skills (`re_analyst/skills/*/SKILL.md`) until the agent achieves a mean score **above 4.0** (achieved **5.0000 / 5.0000**) on both target criteria: `cost-per-acre` and `neighboring-flood-risk`.

---

## Repository Structure

```text
autoraters/
├── README.md                                  # Complete step-by-step runbook (Tasks 1–6) & reusable code guide
├── RUNBOOK.md                                 # Copy of the complete runbook
├── requirements.txt                           # Python 3.13 dependencies (google-adk, google-cloud-aiplatform, krippendorff, etc.)
├── best_criteria_submission.txt               # Task 5 submission: monthly-tax-holding,flood-zone-flagging,neighboring-flood-risk
├── re_analyst/                                # ADK Real Estate Investment Analyst Agent
│   ├── agent.py                               # Root ADK Agent + SkillToolset + FunctionTools definition
│   ├── skills/                                # Refined ADK Skills (Hill-Climbed to 5.0000 Mean Score)
│   │   ├── analyze-opportunity/SKILL.md       # Core parcel + adjacent parcel lookup & Cost-Per-Acre derivation/benchmarking
│   │   ├── analyze-flood-risk/SKILL.md        # Subject & neighboring parcel FEMA flood zone & hydrological/runoff analysis
│   │   └── analyze-tax-implications/SKILL.md  # Tax assessment lookup, monthly tax holding cost, & per-acre valuation basis
│   └── tools/
│       ├── lot_tool.py                        # FunctionTool: get_lot_data(lot_id)
│       └── tax_tool.py                        # FunctionTool: get_tax_assessment_data(parcel_apn)
├── eval_harness/                              # Evaluation & Reliability Analysis Harness
│   ├── create_re_agent_evalset.py             # Batch runner: executes ADK agent concurrently & exports ADK EvalSet JSON
│   ├── run_geap_eval.py                       # Vertex AI Gen AI Eval Service runner (PointwiseMetric + k-pass execution)
│   ├── rubrics.json                           # 19 evaluation rubric criteria (1–5 Likert scales)
│   ├── hillclimb_lots.txt                     # 10 target Lot IDs for baseline and hill-climbing evaluation
│   ├── all_criteria_analysis.py               # Computes Krippendorff's alpha & agreement metrics across all 19 criteria
│   ├── single_criterion_analysis.py           # Detailed per-lot score matrix & agreement stats for a single criterion
│   ├── explain_criteria_results.py            # Extracts autorater explanations/rationales for low-scoring reports
│   ├── metrics_calc.py                        # Statistical implementation of Krippendorff's alpha, pairwise agreement, etc.
│   └── run_hillclimb.py                       # Automated hill-climbing pipeline (runs agent + evaluates k=1 on target criteria)
├── tools_data/                                # Mock County & Tax Databases
│   ├── lots_database.json                     # Property records (acreage, zoning, flood_zone, adjacent_parcels, etc.)
│   └── tax_database.json                      # County tax assessment records keyed by parcel_apn
└── runs/
    └── hillclimb_1/
        └── geap_calls_and_responses_hillclimb_1.json  # Final hill-climbing evaluation output (5.0000 on both criteria)
```

---

## Step-by-Step Lab Execution Guide (Tasks 1 to 6)

### Task 1: Set Up Your Environment

#### 1. Authenticate Cloud Shell with Application Default Credentials (ADC)
```bash
gcloud auth application-default login
```

#### 2. Download the Lab Codebase from Cloud Storage
```bash
gcloud storage cp -r gs://<YOUR_PROJECT_ID>-bucket/autoraters .
```
*(In this lab session: `gs://qwiklabs-gcp-01-88e5afd2a01d-bucket/autoraters`)*

#### 3. Install Python 3.13 via `pyenv` and Install Dependencies
ADK and the evaluation harness use Python 3.13:
```bash
sudo apt-get update
sudo apt-get install -y libreadline-dev

curl https://pyenv.run | bash

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc

pyenv install 3.13
pyenv global 3.13

python3 -m pip install -r autoraters/requirements.txt
```
> **Tip:** If your Cloud Shell session disconnects and reconnects, re-enable Python 3.13 with:
> ```bash
> export PYENV_ROOT="$HOME/.pyenv" && export PATH="$PYENV_ROOT/bin:$PYENV_ROOT/shims:$PATH"
> ```

#### 4. Create the `.env` Configuration File
```bash
cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=qwiklabs-gcp-01-88e5afd2a01d
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF
```
> **Note:** If `gemini-3.5-flash` experiences transient quota/capacity throttling, you can set `MODEL=gemini-3.1-flash-lite` in `.env`.

#### 5. Set the Active GCP Project
```bash
gcloud config set project qwiklabs-gcp-01-88e5afd2a01d
```

---

### Task 2: Run, Test, and Understand the Agent

#### 1. Start the ADK Web UI
From `~/autoraters`, launch the interactive ADK web server:
```bash
cd ~/autoraters
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```
Open the Cloud Shell Web Preview on port `8000`, select the `re_analyst` agent, and test it with a prompt such as:
```text
Analyze LOT-8472
```
Stop the web server with `Ctrl+C` when finished inspecting the trace.

#### 2. How the `re_analyst` Agent Works (`re_analyst/agent.py`)
- **Model**: `Gemini(model=MODEL)` with retry options.
- **Tools**:
  - `get_lot_data(lot_id: str)` (`re_analyst/tools/lot_tool.py`): Queries `tools_data/lots_database.json` and returns parcel attributes (`plot_id`, `parcel_apn`, `acreage`, `zoning`, `flood_zone`, `last_sale_price`, `estimated_annual_taxes`, `environmental_status`, `utility_access`, `topography`, `adjacent_parcels`, `soil_classification`, `easement_type`, etc.).
  - `get_tax_assessment_data(parcel_apn: str)` (`re_analyst/tools/tax_tool.py`): Queries `tools_data/tax_database.json` by `parcel_apn` and returns county tax records (`assessed_land_value`, `assessed_improvement_value`, `total_assessed_value`, `annual_tax_amount`, `tax_rate_area`, `tax_delinquent`).
- **Skills (`SkillToolset`)**:
  - Loaded dynamically from `re_analyst/skills/` using `load_skill_from_dir`:
    1. `analyze-opportunity`
    2. `analyze-flood-risk`
    3. `analyze-tax-implications`
  - **Important ADK Concept**: Each skill's YAML frontmatter defines `metadata.adk_additional_tools` to unlock specific `FunctionTool`s when that skill is loaded by the agent.

---

### Task 3: Generate Baseline Agent Responses (`create_re_agent_evalset.py`)

Run the `re_analyst` agent concurrently across the 10 target lots in `eval_harness/hillclimb_lots.txt` (`LOT-8472`, `LOT-8471`, `LOT-8473`, `LOT-1000` through `LOT-1006`) and export an ADK-compatible `EvalSet` JSON file:

```bash
cd ~/autoraters
python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --output re_analyst/hillclimb_lots.evalset.json \
  --eval-set-id baseline_evalset \
  --agent-dir re_analyst
```

#### What This Script Does Under the Hood
- Loads `re_opportunity_analyst` from `re_analyst/agent.py`.
- Spins up an `InMemoryRunner` session per lot ID (up to 10 concurrent sessions via `asyncio.Semaphore(10)`).
- Sends `"Please conduct a full investment opportunity analysis for lot ID: {lot_id}"`.
- Captures all intermediate tool calls (`IntermediateData`) and the final markdown memorandum (`final_response`), saving them as `EvalCase` entries in `re_analyst/hillclimb_lots.evalset.json`.

---

### Task 4: Evaluate Agent Reports with LLM-as-a-Judge (`run_geap_eval.py`)

#### Step 1: Dry-Run Mode (Prompt Assembly Verification)
Generate the assembled LLM-as-a-Judge evaluation prompts without making live API calls (`--dry-run --k 3`):
```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3
```
- **Output**: Creates `runs/baseline/geap_calls_and_responses.json` with **190 records** (`10 property reports × 19 rubric criteria`).

#### Step 2: Live Evaluation Mode (Optional in Lab)
Removing `--dry-run` dispatches `k` stochastic passes to the Vertex AI Gen AI Evaluation Service (`EvalTask.evaluate()`):
```bash
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses_live.json \
  --experiment-name rubric-agreement-check \
  --k 3
```

---

### Task 5: Analyze Autorater Self-Consistency & Choose Reliable Criteria

Because a full 32-lot × 19-criteria × 15-pass evaluation requires 9,120 judge calls, the lab provides a pre-computed dataset at `runs/geap_calls_and_responses_for_analysis.json`.

#### 1. Run Multi-Criteria and Single-Criterion Statistical Analysis
```bash
cd ~/autoraters

# Summary table across all 19 criteria
python3 -m eval_harness.all_criteria_analysis \
  --input runs/geap_calls_and_responses_for_analysis.json

# Detailed per-lot rating distribution for an individual criterion
python3 -m eval_harness.single_criterion_analysis \
  --input runs/geap_calls_and_responses_for_analysis.json \
  --criterion monthly-tax-holding
```

#### 2. Complete Statistical Results Across All 19 Criteria ($N=32$ lots, $k=15$ passes)

| Criterion ID | Krippendorff's $\alpha$ | Pairwise `%_agree` | `%_reports_unanimous` | `max_range_report` | `sd_report_means` | `overall_mean` | Reliability Verdict |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **`neighboring-flood-risk`** | **1.000** | **100.00%** | **100.00%** | **0** | **0.1768** | 1.06 | **Reliable (Selected #1)** |
| **`monthly-tax-holding`** | **0.997** | **99.17%** | **93.75%** | **1** | **0.8872** | 2.68 | **Reliable (Selected #2)** |
| **`flood-zone-flagging`** | **0.897** | **94.41%** | **71.88%** | **1** | **0.3357** | 4.89 | **Reliable (Selected #3)** |
| `environmental-esa` | 0.897 | 79.10% | 37.50% | 3 | 0.7798 | 4.42 | High $\alpha$, but wide range (`3`) & lower exact agreement |
| `risk-assessment` | 0.803 | 85.56% | 59.38% | 2 | 0.3694 | 4.65 | Borderline ($\alpha = 0.803$, range `2`) |
| `utility-topography` | 0.728 | 66.22% | 12.50% | 2 | 0.5624 | 3.74 | Moderate/Tentative ($0.67 \le \alpha < 0.80$) |
| `document-formatting` | 0.659 | 89.02% | 62.50% | 2 | 0.2517 | 4.07 | Unreliable ($\alpha < 0.67$) |
| `cost-per-acre` | 0.565 | 86.25% | 65.62% | 3 | 0.3127 | 1.15 | Noisy in baseline ($\alpha = 0.565$, range `3`) |
| `core-specifications` | 0.562 | 94.44% | 75.00% | 2 | 0.1455 | 4.95 | Low variance / low $\alpha$ |
| `data-density` | 0.498 | 99.58% | 96.88% | 1 | 0.0884 | 3.98 | Near-zero variance distractor |
| `professional-tone` | 0.498 | 99.58% | 96.88% | 1 | 0.0884 | 4.98 | Near-zero variance distractor |
| `persuasiveness` | 0.273 | 73.12% | 37.50% | 1 | 0.2529 | 4.19 | Unreliable / subjective |
| `analyst-insight` | 0.130 | 76.67% | 31.25% | 2 | 0.1947 | 3.84 | Unreliable / subjective |
| `report-length` | 0.107 | 50.83% | 3.12% | 2 | 0.2442 | 3.55 | Unreliable / vague rubric |
| `vocabulary-choice` | 0.056 | 79.44% | 34.38% | 1 | 0.1592 | 3.88 | Unreliable / subjective |
| `general-tone` | `nan` | 100.00% | 100.00% | 0 | **0.0000** | 5.00 | **Homogeneity Trap** (all 5s, zero variance) |
| `logical-flow` | `nan` | 100.00% | 100.00% | 0 | **0.0000** | 5.00 | **Homogeneity Trap** (all 5s, zero variance) |
| `professionalism` | `nan` | 100.00% | 100.00% | 0 | **0.0000** | 5.00 | **Homogeneity Trap** (all 5s, zero variance) |
| `readability` | `nan` | 100.00% | 100.00% | 0 | **0.0000** | 5.00 | **Homogeneity Trap** (all 5s, zero variance) |

#### 3. Why the "Homogeneity Trap" Criteria Must Be Rejected
- `general-tone`, `logical-flow`, `professionalism`, and `readability` show `100.00%` pairwise agreement and `0` range **only because the judge gave a score of `5` to every single report across all 15 passes** (`sd_report_means = 0.0000`).
- Without variance across reports, **Krippendorff's Alpha** ($\alpha = 1 - \frac{D_o}{D_e}$) has expected disagreement $D_e = 0$, making $\alpha$ undefined (`NaN`). We have zero evidence the autorater can distinguish a poor report from a good one on these criteria.
- Conversely, **`monthly-tax-holding`** ($\alpha = 0.997$), **`flood-zone-flagging`** ($\alpha = 0.897$), and **`neighboring-flood-risk`** ($\alpha = 1.000$) have both $\alpha \ge 0.80$, non-zero variance across reports (`sd_report_means > 0`), high pairwise agreement (`> 94%`), and tight pass-to-pass score range (`max_range_report <= 1`).

#### 4. Save and Upload the Task 5 Submission (`best_criteria_submission.txt`)
```bash
cd ~/autoraters
echo -n "monthly-tax-holding,flood-zone-flagging,neighboring-flood-risk" > best_criteria_submission.txt
gcloud storage cp ~/autoraters/best_criteria_submission.txt gs://qwiklabs-gcp-01-88e5afd2a01d-bucket/best_criteria_submission.txt
```

---

### Task 6: Perform Hill-Climbing Automation (`run_hillclimb.py`)

#### 1. Diagnose Agent Failures Using Autorater Explanations (`explain_criteria_results.py`)
```bash
cd ~/autoraters

# Inspect why cost-per-acre scored poorly (~1.15 mean)
python3 -m eval_harness.explain_criteria_results \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion cost-per-acre \
  --max_score_to_show 3.0

# Inspect why neighboring-flood-risk scored poorly (~1.06 mean)
python3 -m eval_harness.explain_criteria_results \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion neighboring-flood-risk \
  --max_score_to_show 3.0
```

#### 2. Root Cause Analysis of Baseline Deficiencies
1. **`cost-per-acre` (Rubric Target for Score 5: *"Accurately calculates and highlights exact cost-per-acre as a core valuation metric"*):**
   - **Problem**: The agent reported `last_sale_price` and `acreage` separately but never divided `Last Sale Price / Acreage`, never showed the mathematical derivation formula, and never highlighted or benchmarked **Acquisition Cost-Per-Acre** as a core valuation metric.
2. **`neighboring-flood-risk` (Rubric Target for Score 5: *"Exhaustively investigates neighboring property flood zones and critically analyzes potential drainage, runoff, and shared hydrological impacts on the subject parcel's development feasibility"*):**
   - **Problem**: Every property record in `tools_data/lots_database.json` contains an `"adjacent_parcels"` array (e.g., `LOT-8472` has `["LOT-8471", "LOT-8473"]`). However:
     - `analyze-opportunity/SKILL.md` only instructed the agent to call `get_lot_data` once for the subject lot.
     - `analyze-flood-risk/SKILL.md` did not include `get_lot_data` in its `metadata.adk_additional_tools` frontmatter and never instructed the agent to look up the `adjacent_parcels` or analyze cross-boundary drainage, topographic slope gradients, soil permeability, or floodplain contiguity.

#### 3. Refined Skill Files (`re_analyst/skills/*/SKILL.md`)

##### `re_analyst/skills/analyze-opportunity/SKILL.md`
```markdown
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
```

##### `re_analyst/skills/analyze-flood-risk/SKILL.md`
```markdown
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
```

##### `re_analyst/skills/analyze-tax-implications/SKILL.md`
```markdown
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
```

#### 4. Run the Hill-Climbing Script (`run_hillclimb.py`)
```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```

#### 5. Final Hill-Climbing Output (`hillclimb_1`)
```text
✅ Evaluation results saved to -> /home/student_03_982f90e5ef72/autoraters/runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json

=========================================
📊 Hill Climbing Results: hillclimb_1
=========================================
 • cost-per-acre             Mean Score: 5.0000
 • neighboring-flood-risk    Mean Score: 5.0000
=========================================
```

---

## Key Reusable Code Patterns for Future Reference

### 1. Defining an ADK Agent with `SkillToolset` and `FunctionTool` (`re_analyst/agent.py`)
```python
from google.adk.agents import Agent
from google.adk.models import Gemini
from google.adk.skills import load_skill_from_dir
from google.adk.tools import FunctionTool
from google.adk.tools.skill_toolset import SkillToolset

lot_tool = FunctionTool(func=get_lot_data)
tax_tool = FunctionTool(func=get_tax_assessment_data)

opportunity_skill = load_skill_from_dir("re_analyst/skills/analyze-opportunity")
flood_skill = load_skill_from_dir("re_analyst/skills/analyze-flood-risk")
tax_skill = load_skill_from_dir("re_analyst/skills/analyze-tax-implications")

skill_toolset = SkillToolset(
    skills=[opportunity_skill, flood_skill, tax_skill],
    additional_tools=[lot_tool, tax_tool],
)

agent = Agent(
    name="re_opportunity_analyst",
    model=Gemini(model="gemini-3.5-flash"),
    instruction=SYSTEM_INSTRUCTION,
    tools=[skill_toolset],
)
```

### 2. Building Custom Pointwise Autorater Metrics in Vertex AI (`eval_harness/run_geap_eval.py`)
```python
import vertexai
from vertexai.evaluation import EvalTask, PointwiseMetric, PointwiseMetricPromptTemplate

template = PointwiseMetricPromptTemplate(
    criteria={"Acquisition Cost-Per-Acre Calculation": "Derives and states the exact mathematical acquisition cost-per-acre based on the last sale price and acreage."},
    rating_rubric={
        "1": "Cost-per-acre calculation is entirely omitted or mathematically incorrect.",
        "2": "Alludes to land cost relative to size, but does not provide a numerical figure or formula.",
        "3": "Mentions cost-per-acre roughly or without explicit derivation formula.",
        "4": "Correctly computes cost-per-acre with clear derivation, but does not benchmark or emphasize it as a core valuation metric.",
        "5": "Accurately calculates and highlights exact cost-per-acre as a core valuation metric.",
    },
    input_variables=["response"],
)
# Workaround for Vertex AI SDK PromptTemplate initialization bug:
template.variables = template._get_variables()

metric = PointwiseMetric(
    metric="cost-per-acre",
    metric_prompt_template=template,
)

eval_task = EvalTask(
    dataset=dataset_df,  # pandas DataFrame with a 'response' column
    metrics=[metric],
    experiment="hillclimb-experiment",
)
eval_result = eval_task.evaluate(experiment_run_name="pass-1")
```

### 3. Computing Ordinal Krippendorff's Alpha (`eval_harness/metrics_calc.py`)
```python
import krippendorff
import numpy as np

# reliability_data shape: (k_raters_or_passes, n_items_or_reports)
alpha = krippendorff.alpha(
    reliability_data=matrix_k_by_n,
    level_of_measurement="ordinal",
)
```

# BI + AI Analytics Portfolio Project

> **Tableau meets Claude AI** — A end-to-end analytics project that combines interactive BI dashboards with AI-powered root cause analysis. The dashboard shows the **WHAT**. Claude explains the **WHY**.

### Demo

| Part | Description | Link |
|---|---|---|
| Part 1 | Dashboard walkthrough + Tableau Cloud setup | [Watch on Loom](https://www.loom.com/share/59c3c759e81d4b67957078c693c22cdd) |
| Part 2 | Claude MCP connection + live WHY analysis | [Watch on Loom](https://www.loom.com/share/7cb89d0c7b3145939e7356b57ad6cdde) |

---

## About

This project demonstrates a modern BI + AI workflow built entirely on real-world healthcare data. It was designed as a portfolio piece to showcase how a data analyst can go beyond static dashboards and deliver **conversational, context-aware insights** using Claude AI connected to Tableau Cloud via the Model Context Protocol (MCP).

Rather than a hiring manager clicking through charts, they can watch Claude answer questions like:

> *"Why do diabetes patients readmit at 53.6% — nearly 7 points above the average? Break it down by age, medication status, comorbidities, and prior utilisation."*

...and receive a structured, data-backed root cause analysis in seconds.

---

## Overview

| Item | Detail |
|---|---|
| Dataset | Hospital Readmissions (Kaggle — Option B) |
| Records | 25,000 patients |
| Original columns | 17 |
| Engineered columns | 33 (after feature engineering) |
| Dashboard | Single-tab, 7 charts + 4 KPI cards |
| AI layer | Claude Sonnet via Tableau MCP Server |
| Business questions answered | 8 |

---

## The Problem

Traditional BI dashboards answer **WHAT** is happening:
- Diabetes patients readmit at 53.6%
- High prior-utilisation patients readmit at 78.6%
- Very High Risk tier costs $26,140 avg per patient

But they cannot answer **WHY** it is happening without an analyst manually slicing across every dimension. This project solves that gap by connecting Claude AI directly to the published Tableau data source — so any stakeholder can ask a natural language question and receive a multi-dimensional root cause analysis instantly.

---

## Project Structure

```
hospital-readmissions-bi-ai/
│
├── data/
│   ├── hospital_readmissions.csv                  # Raw Kaggle dataset (Option B)
│   └── hospital_readmissions_tableau_ready.csv    # Cleaned + engineered dataset
│
├── notebooks/
│   └── hospital_readmissions_data_prep.ipynb      # Step-by-step data prep notebook
│
├── scripts/
│   └── feature_engineering.py                     # Python script version of the pipeline
│
├── tableau/
│   └── Hospital_Readmissions_Dashboard.twb        # Tableau workbook (points to Cloud DS)
│
├── docs/
│   └── tableau_dashboard_spec.docx                # Full chart-by-chart build spec
│
├── prompts/
│   └── claude_mcp_prompts.md                      # Pre-built WHY question templates
│
└── README.md
```

---

## Architecture Highlights

```
Raw CSV (Kaggle)
      │
      ▼
Python Feature Engineering
      │  • 17 → 33 columns
      │  • Risk scores, cost estimates
      │  • Clinical dimension engineering
      ▼
Tableau Desktop
      │  • 7 calculated fields
      │  • Single-tab dashboard (1400×900px)
      │  • Dual-axis background bars
      │  • Fixed colour palette
      ▼
Tableau Cloud (Published Data Source)
      │  • Data source published separately
      │  • Workbook points to cloud DS
      │  • PAT authentication enabled
      ▼
Claude MCP Server (local)
      │  • npx @tableau/mcp-server@latest
      │  • Reads claude_desktop_config.json
      │  • Authenticates via PAT token
      ▼
Claude Desktop (AI brain)
      │  • Queries Tableau Cloud directly
      │  • Slices across all dimensions
      └─ Root cause analysis + WHY insights
```

---

## The Data

### Source
- **Dataset:** Hospital Readmissions (Kaggle — dubradave/hospital-readmissions)
- **Origin:** Real-world clinical dataset based on UCI ML Repository diabetic patient records
- **Size:** 25,000 rows, 17 original columns

### Original columns
| Column | Type | Description |
|---|---|---|
| `age` | String | Age bracket e.g. [60-70) |
| `time_in_hospital` | Integer | Length of stay in days |
| `n_lab_procedures` | Integer | Number of lab tests |
| `n_procedures` | Integer | Number of medical procedures |
| `n_medications` | Integer | Number of medications prescribed |
| `n_outpatient` | Integer | Prior outpatient visits |
| `n_inpatient` | Integer | Prior inpatient visits |
| `n_emergency` | Integer | Prior emergency visits |
| `medical_specialty` | String | Treating specialty |
| `diag_1` | String | Primary diagnosis category |
| `diag_2` | String | Secondary diagnosis |
| `diag_3` | String | Tertiary diagnosis |
| `glucose_test` | String | Glucose test result |
| `A1Ctest` | String | HbA1c test result |
| `change` | String | Medication change flag |
| `diabetes_med` | String | On diabetes medication |
| `readmitted` | String | Readmission outcome (yes/no) |

---

## Data Cleaning & Feature Engineering

All transformation is done in `scripts/feature_engineering.py`. Below are the key steps with code snippets.

### Step 1 — Age group standardisation

```python
age_bucket_map = {
    '[40-50)': '40–49', '[50-60)': '50–59', '[60-70)': '60–69',
    '[70-80)': '70–79', '[80-90)': '80–89', '[90-100)': '90+'
}
age_midpoint_map = {
    '[40-50)': 45, '[50-60)': 55, '[60-70)': 65,
    '[70-80)': 75, '[80-90)': 85, '[90-100)': 95
}
age_risk_map = {
    '[40-50)': 'Moderate', '[50-60)': 'Moderate', '[60-70)': 'High',
    '[70-80)': 'High', '[80-90)': 'Very High', '[90-100)': 'Very High'
}

df['age_group']     = df['age'].map(age_bucket_map)
df['age_midpoint']  = df['age'].map(age_midpoint_map)
df['age_risk_tier'] = df['age'].map(age_risk_map)
```

### Step 2 — Readmission binary flag

```python
df['readmitted_flag']  = (df['readmitted'] == 'yes').astype(int)
df['readmitted_label'] = df['readmitted'].str.capitalize()
```

### Step 3 — Length of stay bucket

```python
bins   = [0, 2, 4, 7, 14]
labels = ['1–2 days', '3–4 days', '5–7 days', '8–14 days']
df['los_bucket'] = pd.cut(
    df['time_in_hospital'], bins=bins, labels=labels, right=True
)
```

### Step 4 — Clinical complexity score (composite)

```python
df['complexity_score'] = (
    (df['n_medications']    / df['n_medications'].max()   ) * 0.30 +
    (df['n_lab_procedures'] / df['n_lab_procedures'].max()) * 0.20 +
    (df['n_procedures']     / df['n_procedures'].max()    ) * 0.20 +
    (df['n_inpatient']      / df['n_inpatient'].max()     ) * 0.15 +
    (df['n_emergency']      / df['n_emergency'].max()     ) * 0.15
).round(3)

complexity_labels = pd.cut(
    df['complexity_score'],
    bins=[0, 0.25, 0.50, 0.75, 1.01],
    labels=['Low', 'Moderate', 'High', 'Very High'],
    right=False
)
df['complexity_tier'] = complexity_labels
```

### Step 5 — Diabetes management status

```python
def diabetes_status(row):
    on_med    = row['diabetes_med'] == 'yes'
    changed   = row['change'] == 'yes'
    a1c_high  = row['A1Ctest'] == 'high'
    gluc_high = row['glucose_test'] == 'high'
    if not on_med:
        return 'Not on Medication'
    if on_med and changed and (a1c_high or gluc_high):
        return 'Poorly Controlled'
    if on_med and not changed:
        return 'Stable'
    return 'Adjusting'

df['diabetes_mgmt_status'] = df.apply(diabetes_status, axis=1)
```

### Step 6 — Readmission risk score

```python
df['readmission_risk_score'] = (
    (df['n_inpatient']   > 0).astype(int) * 2 +
    (df['n_emergency']   > 0).astype(int) * 2 +
    (df['n_medications'] > 15).astype(int) * 1 +
    (df['age_midpoint']  >= 70).astype(int) * 1 +
    (df['has_comorbidity'] == 'Yes').astype(int) * 1 +
    (df['diabetes_med']  == 'yes').astype(int) * 1
)

risk_labels = pd.cut(
    df['readmission_risk_score'],
    bins=[-1, 1, 3, 5, 10],
    labels=['Low Risk', 'Moderate Risk', 'High Risk', 'Very High Risk']
)
df['readmission_risk_tier'] = risk_labels
```

### Step 7 — Synthetic cost estimate (for storytelling)

```python
import numpy as np
np.random.seed(42)

base_cost = (
    df['time_in_hospital'] * 1800 +
    df['n_lab_procedures'] * 120  +
    df['n_procedures']     * 3500 +
    df['n_medications']    * 85
)
noise = np.random.normal(1.0, 0.12, len(df))
df['estimated_cost_usd'] = (base_cost * noise).round(0).astype(int)
```

### Final engineered columns (33 total)

| Category | New columns |
|---|---|
| Demographics | `age_group`, `age_midpoint`, `age_risk_tier` |
| Clinical | `primary_diagnosis`, `comorbidity_count`, `has_comorbidity`, `specialty_clean` |
| Diabetes | `diabetes_mgmt_status` |
| Stay | `los_bucket` |
| Medication | `medication_burden` |
| Utilisation | `prior_utilisation`, `prior_utilisation_tier` |
| Complexity | `complexity_score`, `complexity_tier` |
| Risk | `readmission_risk_score`, `readmission_risk_tier` |
| Cost | `estimated_cost_usd` |
| Target | `readmitted_flag`, `readmitted_label` |

---

## How It Works

### The WHAT (Tableau Dashboard)

A single-tab dashboard (1400×900px) with 4 KPI cards and 7 charts:

| Chart | Question answered |
|---|---|
| Readmission rate by primary diagnosis | Which conditions drive the most readmissions? |
| Readmission rate by age group | Which patient age segments are highest risk? |
| Readmission rate by specialty | Which departments have care gaps? |
| Diabetes management status bar | How does diabetes control affect outcomes? |
| Cost donut: readmitted vs not | What is the financial impact? |
| Prior utilisation tier bar | Does prior ER/inpatient use predict readmission? |
| Risk tier funnel | Who are the highest-risk patients? |

### The WHY (Claude MCP)

When a stakeholder sees a metric they want to understand, they fire a pre-built prompt to Claude Desktop. Claude queries the Published Data Source on Tableau Cloud via MCP and returns a multi-dimensional root cause analysis.

**Example prompt:**
```
The Tableau dashboard shows diabetes patients have a 53.6% readmission
rate, nearly 7 points above the 47% average. Using the hospital
readmissions dataset, why is this happening? Break down by age group,
medication status, comorbidities, and prior utilisation.
```

**Claude then:**
1. Calls the Tableau MCP server tools
2. Queries the Published Data Source on Tableau Cloud
3. Slices across age group, diagnosis, medication status, prior utilisation
4. Returns a structured root cause analysis with supporting data

---

## Tech Stack

| Layer | Technology |
|---|---|
| Data preparation | Python, pandas, numpy |
| Dashboard | Tableau Desktop 2024+ |
| Cloud hosting | Tableau Cloud (14-day trial / paid) |
| AI model | Claude Sonnet (Anthropic) |
| AI-to-data bridge | Tableau MCP Server (`@tableau/mcp-server`) |
| MCP runtime | Node.js v20+ |
| AI interface | Claude Desktop (Windows / Mac) |
| Authentication | Tableau Personal Access Token (PAT) |

---

## Setup & Replication

Follow these steps exactly in order to replicate this project.

### Prerequisites
- Python 3.9+
- Tableau Desktop (free trial at tableau.com)
- Tableau Cloud account (free 14-day trial at tableau.com/trial)
- Claude Desktop installed (claude.ai/download)
- Node.js v20+ (nodejs.org — download LTS version)

---

### Part 1 — Data preparation

**1. Download the dataset**

Go to: `kaggle.com/datasets/dubradave/hospital-readmissions`

Download `hospital_readmissions.csv`

**2. Install Python dependencies**

```bash
pip install pandas numpy
```

**3. Run the feature engineering script**

```bash
python scripts/feature_engineering.py
```

This produces `hospital_readmissions_tableau_ready.csv` with 33 columns ready for Tableau.

---

### Part 2 — Tableau Desktop

**1. Connect the data**
- Open Tableau Desktop
- Connect → Text File → select `hospital_readmissions_tableau_ready.csv`

**2. Create calculated fields**

Go to Analysis → Create Calculated Field for each of these:

```
Name: Readmission Rate
Formula: SUM([readmitted_flag]) / COUNT([patient_id])
Format: Percentage, 1 decimal place
```

```
Name: Avg Cost
Formula: AVG([estimated_cost_usd])
Format: Currency USD, 0 decimal places
```

```
Name: Background Bar
Formula: 1.0
(Used as dual-axis grey track behind each bar)
```

```
Name: Risk Tier Sort
Formula:
CASE [readmission_risk_tier]
  WHEN 'Low Risk'       THEN 1
  WHEN 'Moderate Risk'  THEN 2
  WHEN 'High Risk'      THEN 3
  WHEN 'Very High Risk' THEN 4
  ELSE 0
END
```

**3. Build the dashboard**

Refer to `docs/tableau_dashboard_spec.docx` for the complete chart-by-chart specification including shelf configurations, colour hex codes, axis ranges, and validated data values.

**Dashboard colour palette:**

| Use | Hex |
|---|---|
| High / alert (>50% rate) | `#C0392B` |
| Elevated (47–50%) | `#D85A30` |
| Moderate (43–47%) | `#BA7517` |
| Good / low (<43%) | `#2E7D32` |
| Dashboard title | `#1B3A5C` |
| Section headings | `#2E75B6` |
| Claude MCP accent | `#534AB7` |
| Row background | `#F1EFE8` |

---

### Part 3 — Tableau Cloud

**1. Sign up for Tableau Cloud trial**

Go to `tableau.com/trial` and create your account. Note your site URL and site name from the browser:
```
https://prod-us-a.online.tableau.com/#/site/YOURSITENAME/home
                                                ^^^^^^^^^^^^
                                                This is your SITE_NAME
```

**2. Enable Personal Access Tokens**

- Go to Settings → Authentication tab
- Find the Personal Access Tokens section
- Enable: "Allow users to create personal access tokens"
- Save

**3. Generate your PAT**

- Click your profile icon (top right) → My Account Settings
- Scroll to Personal Access Tokens
- Token name: `claude-mcp`
- Click Create New Token
- **Copy the secret immediately — it is only shown once**

Save these four values:
```
SERVER:    https://prod-us-a.online.tableau.com
SITE_NAME: yoursitename
PAT_NAME:  claude-mcp
PAT_VALUE: your-secret-here
```

**4. Publish the data source**

In Tableau Desktop:
- Server → Sign In → enter your Tableau Cloud URL
- Server → Publish Data Source
- Name: `Hospital Readmissions`
- Permissions: All users can view and connect
- Click Publish

**5. Publish the workbook**

- Server → Publish Workbook
- Name: `Hospital Readmissions — BI+AI Dashboard`
- Under Data Sources → set to **Published separately** (not embedded)
- Click Publish

---

### Part 4 — Claude MCP Server

**1. Verify Node.js is installed**

```bash
node -v   # Should return v20.x.x or higher
npm -v    # Should return 10.x.x or higher
```

**2. Find your Claude Desktop config file**

On Windows:
```
%APPDATA%\Claude\claude_desktop_config.json
```

If the standard path does not work, search for the file:
```
C:\Users\YourUsername\AppData\Local\Packages\Claude_xxxxx\LocalCache\Roaming\Claude\
```

On Mac:
```
~/Library/Application Support/Claude/claude_desktop_config.json
```

**3. Edit the config file**

Open `claude_desktop_config.json` in a text editor and paste:

```json
{
  "mcpServers": {
    "tableau": {
      "command": "npx",
      "args": ["-y", "@tableau/mcp-server@latest"],
      "env": {
        "SERVER":    "https://prod-us-a.online.tableau.com",
        "SITE_NAME": "yoursitename",
        "PAT_NAME":  "claude-mcp",
        "PAT_VALUE": "your-secret-here"
      }
    }
  }
}
```

Replace the four `env` values with your actual credentials from Part 3.

**4. Restart Claude Desktop**

- Right-click Claude in the system tray → Quit
- Reopen Claude Desktop from Start menu

**5. Verify the connection**

- Click the `+` button in the chat input
- Select Connectors
- You should see **Tableau MCP Server** toggled ON

**6. Test the connection**

Type this in a new Claude Desktop conversation:

```
What published data sources are available on my Tableau Cloud site?
```

If Claude returns `Hospital Readmissions` by name, the full pipeline is working.

---

## Current Capabilities

Once the pipeline is live, Claude can:

| Capability | Example prompt |
|---|---|
| List data sources | "What published data sources are on my Tableau site?" |
| Query aggregated data | "What is the average readmission rate by diagnosis?" |
| Slice across dimensions | "Break down readmission rate by age group and specialty" |
| Root cause analysis | "Why do diabetes patients readmit at 53.6%?" |
| Risk profiling | "What do Very High Risk patients have in common?" |
| Cost analysis | "What is the total estimated cost of readmitted patients by diagnosis?" |
| Trend identification | "Which patient segment has the sharpest improvement opportunity?" |

---

## Pre-built WHY Prompts

Copy any of these directly into Claude Desktop after connecting MCP:

**Prompt 1 — Diabetes spike**
```
The Tableau dashboard shows diabetes patients have a 53.6% readmission
rate, nearly 7 points above the 47% average. Using the hospital
readmissions dataset, why is this happening? Break down by age group,
medication status, comorbidities, and prior utilisation.
```

**Prompt 2 — Elderly readmission**
```
Patients aged 80–89 have the highest readmission rate at 49.6%. Why
does this age group outperform even 90+ patients? What combination of
clinical, medication, and prior utilisation factors explains this?
```

**Prompt 3 — High prior utilisation**
```
Patients with high prior utilisation readmit at 78.6% — more than
double the rate of low-utilisation patients. What diagnoses and age
groups make up this cohort?
```

**Prompt 4 — Very High Risk profile**
```
The Very High Risk tier has 3,192 patients with a 63.5% readmission
rate. What is the most common combination of age, diagnosis, medication
burden, and prior utilisation in this segment? Give me a plain-English
profile of a typical Very High Risk patient.
```

---

## What Can Be Added

| Enhancement | Description |
|---|---|
| Predictive model | Train an XGBoost classifier on the dataset and embed risk scores |
| Streamlit front end | Build a web app so non-technical users can fire WHY questions without Claude Desktop |
| Real-time data | Connect to a live EHR database instead of static CSV |
| Email alerts | Trigger automated root cause reports when readmission rate crosses a threshold |
| Additional datasets | Add CMS Hospital Readmissions Reduction Program data for benchmarking |
| Multi-sector version | Replicate the architecture with an energy or financial dataset |
| LLM comparison | Compare Claude vs GPT-4o vs Gemini on the same WHY questions |
| Voice interface | Add speech-to-text so clinicians can ask questions hands-free |

---


## License

This project is for portfolio and educational purposes. The dataset is sourced from Kaggle under its original license. No patient data is real — the original dataset is anonymised and the cost field is synthetically generated.

---

<div align="center">

Built with Tableau + Claude AI — demonstrating that the future of BI is not just dashboards, but conversations.

**[Tableau](https://tableau.com)** · **[Claude AI](https://claude.ai)** · **[Tableau MCP Server](https://github.com/tableau/tableau-mcp)**

</div>

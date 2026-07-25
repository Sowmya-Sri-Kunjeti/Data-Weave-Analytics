# DataWeave Analytics

**Wind Turbine SCADA Anomaly Detection & Alerting Pipeline**

An end-to-end, fully automated pipeline that ingests raw wind turbine SCADA data, cleans it, detects anomalies using rule-based thresholds, sends LLM-summarized email alerts, and visualizes turbine health through live Power BI dashboards.

Built as a bootcamp capstone project on the **Kelmarsh 3** turbine dataset (Senvion MM92, 10-minute interval readings).

---

## 🎯 The Problem

Wind turbines generate a constant stream of sensor readings. Without monitoring, developing faults — overheating, performance drift, grid or mechanical issues — often go undetected until real damage or downtime has already occurred.

This project automates that entire loop: clean the raw data, detect anomalies early with rule-based thresholds, alert the team the moment something critical happens, and give stakeholders a live dashboard to track turbine health.

---

## 🏗️ Architecture

One **Master Orchestrator** workflow chains five sub-workflows end to end, triggered with a single click:

```
Master Orchestrator
   │
   ├── WF1 — Data Ingestion
   ├── WF2 — Data Cleaning
   ├── WF3 — SQL Load
   ├── WF4 — Anomaly Detection
   └── WF5 — Email Alerts
```

| Workflow | Purpose | Output |
|---|---|---|
| **WF1 — Ingestion** | Reads the raw Kelmarsh CSV; strips 10 metadata comment lines via a JS Code node before extraction | 4,176 raw rows |
| **WF2 — Cleaning** | Hard-rejects rows with missing timestamps or physically impossible values; renames columns to snake_case; parses timestamps; deduplicates; forward-fills nulls | 4,066 clean rows (110 rejected) |
| **WF3 — SQL Load** | Writes cleaned data into Neon Postgres (`wind_turbine_data_v2`) using a TRUNCATE-before-INSERT pattern for idempotent full refresh | — |
| **WF4 — Anomaly Detection** | Applies rule-based thresholds across four categories — Performance, Thermal, Grid, Mechanical — each with Warning/Critical tiers | Writes to `Turbine_Anomaly_Log_v2` (~1,933 NORMAL / 1,658 CRITICAL / 475 WARNING) |
| **WF5 — Email Alerts** | Pulls the latest batch (last 18 rows ≈ one 3-hour Power BI refresh cycle), groups by severity, and uses a Groq LLM call to write a natural-language summary sentence before sending a formatted email alert | Gmail draft/send |

---

## 🛠️ Tech Stack

| Tool | Role |
|---|---|
| **n8n** | Workflow orchestration |
| **Neon (Serverless Postgres)** | Data storage — cleaned telemetry + anomaly log |
| **Power BI / Power BI Service** | Dashboards, published with 3-hour scheduled refresh |
| **Groq LLM (Llama 3.3 70B)** | Generates the plain-English anomaly summary sentence in each alert email |
| **Gmail (via n8n)** | Delivers formatted alert emails |

---

## 📊 Dashboards

Two dashboards, deliberately no more — built on a join of the cleaned telemetry and anomaly log tables.

**Performance Dashboard**
- Power curve: wind speed vs. power output, color-coded by anomaly status
- Drill-through page (**Performance Trends**): capacity factor and actual-vs-potential power output over a single day

**Anomaly Dashboard**
- Total anomalies, Critical/Warning counts, breakdown by type
- Condition-monitoring overlay: power output over time with anomaly points plotted directly on the curve
- Drill-through to a full **Anomaly Log**: every flagged reading with its reason and recommended action

Published to Power BI Service with automatic refresh every 3 hours — decoupled from the n8n pipeline, with WF5's alert email covering urgency in between refreshes.

---

## 🧠 Key Engineering Decisions

- **Rule-based, not ML** — anomaly detection uses deterministic thresholds (`Anomaly_Detection_Logic.docx`) rather than a black-box model, so results are explainable and reproducible.
- **AI used narrowly** — the Groq LLM call in WF5 is the pipeline's *only* AI integration point, and it's scoped to writing one summary sentence. All grouping and formatting logic stays deterministic — complexity should be earned, not decorative.
- **Scheduled refresh over event-triggered** — a direct Power BI REST API refresh was evaluated and ruled out (blocked by tenant-admin permissions); Power Automate was viable but required a paid license. Scheduled refresh (every 3 hours) was chosen as the simplest path that met the project's real needs.
- **TRUNCATE-before-INSERT** — used in both WF3 and WF4 for idempotent, full-refresh loads appropriate to this static dataset.

---

## 📁 Repository Structure

```
DataWeave-Analytics/
├── README.md
├── workflows/              # n8n workflow exports (JSON)
│   ├── Master_Orchestrator.json
│   ├── WF1_Ingestion.json
│   ├── WF2_Cleaning.json
│   ├── WF3_SQL_Load.json
│   ├── WF4_Anomaly_Detection.json
│   └── WF5_Email_Alerts.json
├── docs/
│   ├── Anomaly_Detection_Logic.docx
│   └── DataWeave_Analytics_Overview.pptx
└── dashboards/
    └── screenshots/
```

---

## 🎥 Demo Videos

https://app.clipchamp.com/consumer/editor/?driveId=E980843824AE6AB4&itemId=E980843824AE6AB4%21s47d23e62d4d6412abee8f4dce40e4cb9&folderId=E980843824AE6AB4%21s4d244287850046618ece28c7add497ce

## 👥 Team

**Sowmya Sri** (Team Lead) · Debasree · Anju Abhilash · Rekha Gattu · Shimnamol · Sumitra

---

## 📌 Dataset

Built on the [Kelmarsh Wind Farm](https://zenodo.org/records/5841834) open SCADA dataset (Kelmarsh 3, Senvion MM92 turbine), publicly released for research and educational use.

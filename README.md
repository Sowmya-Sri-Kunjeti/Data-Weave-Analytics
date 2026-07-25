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

One **Master Orchestrator** workflow chains five sub-workflows end to end, triggered with a single click: Ingestion → Cleaning → SQL Load → Anomaly Detection → Email Alerts.

## 🏗️ Pipeline in Action

| Workflow | Purpose | Output |
|---|---|---|
| **WF1 — Ingestion** | Reads the raw Kelmarsh CSV; strips 10 metadata comment lines via a JS Code node before extraction | 4,176 raw rows |
| **WF2 — Cleaning** | Hard-rejects rows with missing timestamps or physically impossible values; renames columns to snake_case; parses timestamps; deduplicates; forward-fills nulls | 4,066 clean rows (110 rejected) |
| **WF3 — SQL Load** | Writes cleaned data into Neon Postgres (`wind_turbine_data_v2`) using a TRUNCATE-before-INSERT pattern for idempotent full refresh | — |
| **WF4 — Anomaly Detection** | Applies rule-based thresholds across four categories — Performance, Thermal, Grid, Mechanical — each with Warning/Critical tiers | Writes to `Turbine_Anomaly_Log_v2` (~1,933 NORMAL / 1,658 CRITICAL / 475 WARNING) |
| **WF5 — Email Alerts** | Pulls the latest batch (last 18 rows ≈ one 3-hour Power BI refresh cycle), groups by severity, and uses a Groq LLM call to write a natural-language summary sentence before sending a formatted email alert | Gmail draft/send |

Below are screenshots of the Master Orchestrator and individual n8n workflows, showing the pipeline running end to end.

**Master Orchestrator — full pipeline run**
![Master Orchestrator](<img width="1721" height="482" alt="Master Workflow" src="https://github.com/user-attachments/assets/23c55a70-b4eb-4339-be28-b9e244b97d45" />
)

**WF1 — Data Ingestion**
![WF1 Ingestion](<img width="1580" height="527" alt="Data Ingestion" src="https://github.com/user-attachments/assets/e6ec59d2-3584-448b-a0b9-811dafc48ac7" />
)

**WF2 — Data Cleaning**
![WF2 Cleaning](<img width="1827" height="541" alt="Data Cleaning   Validation" src="https://github.com/user-attachments/assets/399e800d-6484-4810-b09e-3cfabec6759b" />
)

**WF3 — SQL Load**
![WF3 SQL Load](<img width="1372" height="392" alt="SQL Data load" src="https://github.com/user-attachments/assets/bc7b90a9-0700-44b5-a0ac-a4e59e17d32a" />
)

**WF4 — Anomaly Detection**
![WF4 Anomaly Detection](<img width="1627" height="600" alt="Anomaly Detection" src="https://github.com/user-attachments/assets/8e75ba32-973f-429a-9be7-32c55954a47e" />
)

**WF5 — Email Alerts**
![WF5 Email Alerts](<img width="1780" height="636" alt="Email Alert" src="https://github.com/user-attachments/assets/c674df32-9e69-4922-8c70-aca6dc51de2f" />
)

**Sample of Email alert**
![Email Alert sample](<img width="727" height="662" alt="sample for email draft" src="https://github.com/user-attachments/assets/ce954d79-4028-4f2c-bb10-768d83c12586" />)


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

**Screenshots of Power BI dashboards**
![Performance Dashboard](<img width="1337" height="762" alt="Performance dashboard" src="https://github.com/user-attachments/assets/121ce7e0-5de9-437f-a678-155e33b82460" />
)

![Performance Trends Dashboard](<img width="1345" height="760" alt="Performance Trends dashboard" src="https://github.com/user-attachments/assets/7dbd7249-504d-47d0-a5e5-996d0eecb00e" />
)

![Anomaly Detection](<img width="1627" height="600" alt="Anomaly Detection" src="https://github.com/user-attachments/assets/cd70ca0d-026b-4b15-8845-e504ffe0aabe" />
)

![Anomaly Log dashboard](<img width="1342" height="775" alt="Anomaly Log dashboard" src="https://github.com/user-attachments/assets/d877d2e4-0294-4a11-ba61-57ae92801946" />
)
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
├── screenshots/             # n8n workflow screenshots
│   ├── master_orchestrator.png
│   ├── wf1_ingestion.png
│   ├── wf2_cleaning.png
│   ├── wf3_sql_load.png
│   ├── wf4_anomaly_detection.png
│   └── wf5_email_alerts.png
├── docs/
│   ├── Anomaly_Detection_Logic.docx
│   └── DataWeave_Analytics_Overview.pptx
└── dashboards/
    └── screenshots/         # Power BI dashboard screenshots
```

---

## 🎥 Demo Videos

- **Project Introduction & Master Workflow Overview** — [link]
- **Power BI Dashboards Walkthrough** — [link]

*(Replace with your Loom links once uploaded)*

---

## 👥 Team

**Sowmya Sri** (Team Lead) · Debasree · Anju Abhilash · Rekha Gattu · Shimnamol · Sumitra

---

## 📌 Dataset

Built on the [Kelmarsh Wind Farm](https://zenodo.org/records/5841834) open SCADA dataset (Kelmarsh 3, Senvion MM92 turbine), publicly released for research and educational use.

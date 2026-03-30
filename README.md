# Episode-Based Assessment Compliance Tracker

A synthetic, end-to-end analytics project simulating care management assessment compliance tracking — from enrollment data through SQL logic, KPI calculation, and dashboard visualization.

Built to reflect real-world population health program requirements, using fully synthetic (non-PHI) data.

---

## What This Project Demonstrates

Care management programs require structured assessments at defined intervals to ensure patients receive timely, coordinated care. Tracking whether or not and when those assessments happen is a core compliance function for analytics teams working with care management data.

This project walks through the full analytics lifecycle for that problem:

1. **Synthetic data generation** — A reproducible R script creates realistic member, enrollment, and assessment records
2. **SQL logic** — Progressive query layers handle episode setup, due-date chaining, compliance flagging, and KPI rollup
3. **Business documentation** — Plain-language context explains what is being measured and why the logic is designed the way it is
4. **Dashboard** — A static HTML dashboard visualizes compliance rates, trends, and member-level detail

---

## Project Structure

```
cm-assessment-compliance/
│
├── README.md                        ← You are here
│
├── data/
│   └── generate_synthetic_data.R    ← R script to generate fake member/assessment data
│
├── sql/
│   ├── 01_episode_setup.sql         ← Enrollment and episode identification
│   ├── 02_assessment_flags.sql      ← 30-day initial and annual due-date logic
│   ├── 03_compliance_flags.sql      ← On-time / late / missing status flags
│   └── 04_kpi_rollup.sql            ← Monthly compliance rate aggregation
│
├── docs/
│   ├── business_context.md          ← What this measures and why it matters
│   ├── logic_decisions.md           ← Design choices and tradeoffs explained
│   └── data_dictionary.md           ← Field definitions for the synthetic schema
│
└── dashboard/
    └── compliance_dashboard.html    ← Static dashboard (opens in any browser)
```

---

## The Compliance Problem This Solves

Care management programs typically require:

- **Initial Assessment** — completed within **30 days** of episode enrollment
- **Annual Reassessment** — completed within **1 year** of the prior assessment, with due dates chaining forward from each completion

Tracking this at scale means solving several non-trivial analytical problems:

- How do you calculate a due date when it depends on a *prior* completion date that may or may not exist?
- How do you handle patients with multiple sequential episodes?
- How do you classify a patient who is *in window* vs. *overdue* vs. *non-compliant* (window closed)?
- How do you aggregate individual-level flags into program-level KPIs that trend over time?

This project works through each of those problems in sequence.

---

## Key SQL Concepts Used

| Concept | Where Used |
|---|---|
| `LAG()` + `COALESCE` date chaining | `02_assessment_flags.sql` — due-date calculation |
| `ROW_NUMBER()` deduplication | `01_episode_setup.sql` — latest episode per member |
| Window functions with `PARTITION BY` | Throughout |
| CTE-based layered logic | All SQL files |
| Compliance status CASE logic | `03_compliance_flags.sql` |
| Monthly aggregation + rolling trends | `04_kpi_rollup.sql` |

SQL is written for **SQL Server 2019** syntax. Redshift-compatible variants are noted inline where syntax differs.

---

## How to Run This Project

### 1. Generate synthetic data
```r
# Requires: R 4.x, tidyverse, lubridate
source("data/generate_synthetic_data.R")
# Outputs: members.csv, episodes.csv, assessments.csv
```

### 2. Load data into your database
Load the three CSVs into a SQL Server or Redshift schema of your choice.

### 3. Run SQL in order
Execute `sql/01_episode_setup.sql` through `sql/04_kpi_rollup.sql` in sequence. Each builds on the prior step.

### 4. Open the dashboard
Open `dashboard/compliance_dashboard.html` in any browser. No server required.

---

## About This Project

Built by [Katie Headley](https://www.linkedin.com/in/katieheadley/), RHIA — Senior Population Health Analyst with experience in care management compliance analytics, HEDIS reporting, and population health data infrastructure.

This project uses entirely synthetic data. No PHI, real member records, or proprietary logic from any employer is included.

---

## Status

| Component | Status |
|---|---|
| Synthetic data script | 🔜 In progress |
| SQL: Episode setup | 🔜 In progress |
| SQL: Assessment flags | 🔜 In progress |
| SQL: Compliance flags | 🔜 In progress |
| SQL: KPI rollup | 🔜 In progress |
| Business context doc | ✅ Complete |
| Logic decisions doc | 🔜 In progress |
| Data dictionary | 🔜 In progress |
| Dashboard | 🔜 In progress |

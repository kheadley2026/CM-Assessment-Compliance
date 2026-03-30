# Business Context: Care Management Assessment Compliance

## What Is Care Management?

Care management (CM) is a structured, proactive approach to coordinating healthcare for patients with complex medical, behavioral, or social needs. Rather than waiting for patients to seek care reactively, care management programs identify high-risk or high-need individuals — often through claims data, clinical flags, or risk stratification — and assign them to a care manager (typically a nurse, social worker, or other clinician) who coordinates their care across providers, settings, and services.

Care management programs operate under many names depending on the organization and population served: Complex Care Management, Transitions of Care, Chronic Disease Management, and Enhanced Care Management are common examples. Programs may be run by health plans, accountable care organizations (ACOs), or health systems directly.

---

## What Is a Care Management Episode?

An **episode** is a discrete enrollment period during which a patient is actively participating in a care management program. Key properties:

- An episode has a **start date** (also called intake date or enrollment date), which anchors all assessment timing
- An episode may have an **end date** if the patient is discharged from the program, disenrolls, or is no longer eligible
- A single patient may have multiple episodes over time — sequential (one ends, another begins) or simultaneous (enrolled in both a medical CM program and a behavioral health CM program at the same time)
- Episode status is typically tracked as: Referral → Open → Closed

All assessment compliance logic in this project is anchored to the episode start date and subsequent assessment completion dates.

---

## Why Are Assessments Required?

Comprehensive assessments are the clinical and operational foundation of care management. They serve several purposes:

- **Clinical:** Identify the patient's full picture — medical conditions, medications, functional status, behavioral health needs, social determinants of health (SDOH), and care gaps
- **Care planning:** Provide the information needed to build a personalized care plan with the patient
- **Regulatory:** Many payers (Medicaid, Medicare Advantage plans, ACOs) require documented assessments within defined timeframes as a condition of program participation and reimbursement
- **Quality measurement:** Assessment completion rates are tracked as indicators of program engagement and care quality

Failure to complete assessments on time represents both a clinical risk (the patient's needs go unaddressed) and a compliance risk (the program may not meet contractual or regulatory requirements).

---

## The Two Assessment Types This Project Tracks

### Initial Assessment (30-Day)

The initial assessment is the first comprehensive assessment conducted after a patient enrolls in a care management episode.

**Compliance requirement:** Must be completed within **30 days** of the episode start date.

**Why 30 days?** This window reflects both clinical urgency and regulatory convention. For newly enrolled patients — particularly those transitioning from a hospitalization or flagged as high-risk — a 30-day window ensures timely needs identification before gaps in care widen. Many payer contracts and CMS program requirements (e.g., for ACOs and managed Medicaid) specify this timeframe explicitly.

**Compliance logic:**
```
Compliant     → assessment_date ≤ episode_start_date + 30 days
Pending       → no assessment yet, but today ≤ episode_start_date + 30 days
Non-Compliant → no assessment completed, and today > episode_start_date + 30 days
Completed Late → assessment_date > episode_start_date + 30 days (completed, but outside window)
```

---

### Annual Reassessment

The annual reassessment is a comprehensive reassessment required throughout the episode to ensure the patient's care plan remains current and their needs are being addressed.

**Compliance requirement:** Must be completed within **1 year** of the prior assessment.

**Due-date chaining logic:** This is the most analytically complex part of assessment compliance tracking. The due date for each reassessment is calculated from the *completion date of the prior assessment*, not from a fixed calendar anchor:

```
First reassessment due:       1 year after initial assessment completion date
Second reassessment due:      1 year after first reassessment completion date
Third reassessment due:       1 year after second reassessment completion date
... and so on
```

This means the due-date schedule is unique to each patient and shifts forward whenever a reassessment is completed. A patient who completes their reassessment two months late will have their *next* due date calculated from that late completion — the clock resets from actual completion, not from the original due date.

**Why chaining matters analytically:** A simple "is there an assessment in the last 12 months?" query will produce incorrect results for patients with complex episode histories. The correct logic requires using `LAG()` window functions to identify the prior completion date for each assessment record, then calculating the next due date forward from that anchor.

**Compliance status flags:**

| Status | Definition |
|---|---|
| Compliant | Completed on or before the calculated due date |
| Completed Late | Completed after the due date, but completed |
| Overdue | Due date has passed; no reassessment on record |
| Pending | Due date is in the future; no reassessment yet (not yet non-compliant) |
| Upcoming | Due within the next 30/60/90 days — actionable for outreach |

---

## How Compliance Rates Are Calculated

At the program level, compliance rates are expressed as:

```
Compliance Rate = Compliant Count / Eligible Denominator Count
```

**Denominator:** Active episodes as of the reporting period, excluding patients who are not yet due (initial assessment window still open, or reassessment not yet due). Exclusions may also apply for patients in hospice, LTSS (Long-Term Services and Supports), or other program-specific categories.

**Numerator:** Patients in the denominator who completed their required assessment on time.

Rates are typically reported:
- **Monthly** — point-in-time compliance rate for that month
- **Stratified** — broken out by line of business, program, care management team, or individual care manager

---

## Who Uses This Data?

| Audience | How They Use It |
|---|---|
| Care managers | Know which of their patients are overdue or upcoming — actionable worklist |
| CM supervisors | Monitor team-level compliance; identify coaching opportunities; evaluate staffing needs |
| Program directors | Track program-level KPIs; report to payers or leadership |
| Quality/compliance teams | Document adherence to contractual or regulatory requirements |
| Analytics team | Build and maintain the underlying data infrastructure |

---

## Complexity Factors

A few things make care management compliance analytics more complex than they first appear:

1. **Due dates are dynamic, not fixed.** They chain from completion dates, so a static date lookup table won't work. Logic must be recalculated as new assessments are completed.

2. **Multiple episodes per patient.** A patient's assessment history must be partitioned by episode to avoid cross-episode contamination in window calculations.

3. **Status depends on today's date.** The same record may be "Pending" on one day and "Overdue" the next. Point-in-time vs. as-of-date reporting requires care in how status flags are materialized. Care management data can change quickly and requires a data warehouse updated frequently.

4. **Late completions are not the same as non-completions.** A program may want to distinguish "never assessed" from "assessed late" for different intervention purposes — this requires more granular status flags than a simple binary compliant/non-compliant field.

5. **Operational and regulatory metrics might differ** Operationally, it might make sense to count unsuccessful attempts at reaching the patient as compliant even if they did not complete the assessment. Regulatory metrics may not grant credit for trying. It might be worth tracking it both ways depending on the organization's goals.

This project addresses each of these challenges explicitly in the SQL logic layers. See [`logic_decisions.md`](logic_decisions.md) for design rationale on specific choices.

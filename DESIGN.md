# Care Plan Auto-Generation System — Design Doc

## 1. Background

### 1.1 Customer & Context

- **Customer**: CVS specialty pharmacy
- **Users**: CVS medical staff (pharmacists / medical assistants). **Patients do not interact with this system.**
- **Workflow**: When a pharmacist prescribes a medication, a Care Plan must be generated alongside the order. The medical staff prints the Care Plan and hands it to the patient.

### 1.2 Pain Points

- Pharmacists currently spend **20–40 minutes** per patient writing a Care Plan manually.
- The Care Plan is **mandatory for Medicare and pharma reimbursement / compliance** — it cannot be skipped.
- The team is severely understaffed and the backlog is growing.

### 1.3 Goal

Reduce Care Plan creation time from 20–40 minutes to a few minutes via a web form + LLM workflow, while guaranteeing data integrity and compliance traceability.

---

## 2. Core Concepts

- **One Care Plan = one Order = one medication.** Each submission produces exactly one Care Plan tied to a single drug.
- **Every Care Plan must contain these four sections**:
  1. Problem list / Drug Therapy Problems (DTPs)
  2. Goals (SMART)
  3. Pharmacist interventions / plan
  4. Monitoring plan & lab schedule

---

## 3. User Journey

1. The medical assistant (MA) opens the web form.
2. MA enters patient info, provider, diagnoses, medication, and patient records.
3. The system runs field-level validation and duplicate detection.
4. On success, the system calls an LLM to generate the Care Plan.
5. MA previews and downloads the Care Plan as a text file to hand to the patient.
6. Data is persisted for downstream pharma reporting / export.

---

## 4. Data Model

### 4.1 Input Fields

| Field | Type | Validation |
| --- | --- | --- |
| Patient First Name | string | Required, non-empty |
| Patient Last Name | string | Required, non-empty |
| Patient DOB | date | Required |
| Patient MRN | 6-digit number | Required, unique system-wide |
| Referring Provider Name | string | Required |
| Referring Provider NPI | 10-digit number | Required, unique system-wide |
| Primary Diagnosis | ICD-10 code | Required, format-validated |
| Additional Diagnoses | list of ICD-10 codes | Optional, each format-validated |
| Medication Name | string | Required |
| Medication History | list of strings | Optional |
| Patient Records | string OR PDF | Required (one of the two) |

### 4.2 Entity Relationships

- **Provider**: keyed by NPI; stored exactly once across the system.
- **Patient**: keyed by MRN.
- **Order**: one submission = one Order = one Care Plan; bound to Patient + Medication + submission date.

---

## 5. Validation & Duplicate Detection

### 5.1 Field-Level Validation

- Non-empty / type / format checks (MRN must be 6 digits, NPI must be 10 digits, ICD-10 format).
- PDF MIME type and file-size checks.

### 5.2 Duplicate Detection Rules

| Scenario | Severity | Behavior | Rationale |
| --- | --- | --- | --- |
| Same patient + same medication + **same day** | ❌ ERROR | Block submission | Definitely a duplicate |
| Same patient + same medication + **different day** | ⚠️ WARNING | Allow user to confirm and continue | May be a refill |
| Same MRN + different name or DOB | ⚠️ WARNING | Allow user to confirm and continue | Likely a data-entry error |
| Same name + DOB + different MRN | ⚠️ WARNING | Allow user to confirm and continue | Possibly the same person |
| Same NPI + different provider name | ❌ ERROR | Must be corrected | NPI is the unique provider identifier |

---

## 6. Feature Checklist

| Feature | Required | Notes |
| --- | --- | --- |
| Web form data entry | ✅ | Used by medical assistants |
| Field validation | ✅ | All inputs must pass validation |
| Patient / order duplicate detection | ✅ | Must not disrupt existing workflow |
| Provider duplicate detection | ✅ | Directly affects pharma reporting accuracy |
| LLM-based Care Plan generation | ✅ | Core capability |
| Care Plan text download | ✅ | Users upload it into their own system |
| Data export (pharma reports) | ✅ | Compliance & reimbursement |

---

## 7. System Architecture (Proposed)

```
[Web Form (Frontend)]
        │
        ▼
[Validation Layer] ── field validation + duplicate detection
        │
        ▼
[Domain / Service Layer]
   ├── PatientService
   ├── ProviderService
   ├── OrderService
   └── CarePlanService ── calls LLM
        │
        ▼
[Persistence Layer] ── Patient / Provider / Order / CarePlan
        │
        ▼
[Export Module] ── pharma reports (CSV / Excel)
```

### 7.1 Module Layout

- **schemas/** — Pydantic (or equivalent) models and validation.
- **services/** — Business logic (duplicate detection, Care Plan generation, export).
- **repositories/** — Persistence (database or file storage).
- **llm/** — Prompt templates and LLM client wrapper.
- **api/** — Form intake and HTTP responses.
- **ui/** — Frontend form (Streamlit / React / similar).
- **tests/** — Coverage for validation, duplicate detection, and Care Plan generation.

### 7.2 LLM Module

- **Input**: structured fields + Patient Records (raw text or text extracted from PDF).
- **Prompt**: must enforce the four mandatory sections (Problem list, Goals, Interventions, Monitoring plan).
- **Output**: plain text (easy to print and upload to third-party systems).

---

## 8. Production-Ready Requirements

| Requirement | Implementation |
| --- | --- |
| Every input is validated | Schema-layer + service-layer double validation |
| Integrity rules always enforced | NPI uniqueness, MRN uniqueness, Order uniqueness enforced at both DB and service layer |
| Errors are safe, clear, contained | Distinguish Validation / Business / System errors; messages name the offending field and reason |
| Code is modular and navigable | Layered packages: schemas / services / repositories / llm / api / ui |
| Critical logic covered by automated tests | Validation rules, duplicate detection, and Care Plan structure assertions all tested |
| Project runs end-to-end out of the box | One-command bootstrap (deps, DB, sample data) documented in README |

---

## 9. Export / Reporting

- Filter by date range, provider, or medication.
- Output formats: CSV (minimum) / Excel.
- Includes pharma-required fields: Patient MRN, Provider NPI, Medication, Diagnosis, Care Plan generation timestamp, etc.

---

## 10. Risks & Open Questions

1. **PHI compliance**: Patient data is HIPAA-regulated. The design assumes deployment in a compliant environment; LLM calls must use a compliant channel (e.g., enterprise OpenAI / Bedrock with logging disabled).
2. **LLM output quality**: A pharmacist must review the output. The system should support "edit before download" — v1 may ship download-only, with edit support in v1.1.
3. **PDF parsing**: Scanned PDFs vs. text PDFs require different paths. v1 may require text PDFs only.
4. **Refill detection**: Currently only "different day + same medication" triggers a refill warning; future versions can incorporate dosing-cycle logic.

---

## 11. Milestones (Proposed)

- **M1** — Data model + validation + form intake + duplicate detection
- **M2** — LLM Care Plan generation + download
- **M3** — Pharma report export
- **M4** — Test coverage, documentation, one-command bootstrap

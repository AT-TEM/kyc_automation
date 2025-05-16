# KYC Automation for AFSL Compliance – prd.md

## 1. Problem Statement / Goal

Manual KYC checks for new counterparties are slow (5‑45 min each) and error‑prone, and analysts must manually capture evidence for audits.
The goal for **Phase 1** is to automate public‑data look‑ups, exemption logic, **automatic evidence capture**, and audit documentation inside HubSpot so that average processing time per ticket falls by **≥ 50 %** while maintaining AFSL audit traceability.

## 2. Target Users & Context

| Persona                                      | Needs                                                                       | Environment                                                         |
| -------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **Compliance / Ops Analyst** (non‑technical) | Rapid, reliable KYC decisions; clear audit trail including supporting files | Works entirely in HubSpot; may trigger low‑code Python integrations |
| **Auditor / Regulator**                      | Evidence of correct KYC steps, with source artefacts                        | Reads HubSpot ticket history + follows links to stored evidence     |

*Volume*: \~2 tickets / day (≈ 40 / mo).
*Deployment*: Single‑tenant, low‑volume; no high‑availability requirement.

## 3. Success Metrics

| Metric                                                | Baseline    | Target                 |
| ----------------------------------------------------- | ----------- | ---------------------- |
| Avg. processing time per ticket                       | 5‑45 min    | **≤ 50 % of baseline** |
| % tickets auto‑exempt without analyst edits           | —           | ≥ 80 %                 |
| Audit completeness (decision + evidence files linked) | qualitative | **100 %**              |

## 4. User Stories

* **As an Ops Analyst**, *when a ticket with an ABN is created*, I want the system to automatically fetch licence & listing status so I don’t search manually.
* **As an Ops Analyst**, I want the system to mark “Simplified KYC – UBO check skipped” (with rationale) when exemptions apply.
* **As an Auditor**, I need every decision, API response, **and the source evidence files – ABN lookup payload, AFSL register entry, screenshot of ASX listing or other licence – saved in a folder and linked in the ticket**, so audits are self‑contained.

## 5. Assumptions & Constraints

* Public API access: ABN Lookup (GUID already provisioned), ASIC registers, ASX listings, APRA registry, ABLIS.
* Low‑code preferred (HubSpot workflows + lightweight Python service).
* No paid or biometric checks in Phase 1; Equifax and Stack Go integrations are deferred.
* **Evidence files**:

  * ABN Lookup – raw JSON saved as `.json`.
  * AFSL / licence – HTML converted to PDF or saved HTML snapshot.
  * ASX or other listing – full‑page screenshot (PNG) captured headlessly.
  * Files stored in an external folder (e.g., AWS S3 bucket `kyc-evidence/`) under path `/<ticket‑id>/<abn>/` and linked back to the HubSpot ticket.
* Data retention follows existing HubSpot policy (7 years unless superseded).

## 6. Solution Overview

```
HubSpot Ticket (Created/ABN entered) ─▶ HS Workflow Webhook
                                   │
                                   ▼
                         **KYC Service (Python/Lambda)**
                         • Public API Aggregator
                         • Decision Engine
                         • Evidence Collector & File Storage
                         • HubSpot Notes Writer
                         • (Opt.) GPT Summariser
                                   │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
        Public Registries                External Storage (S3)
   (ABN, ASIC, ASX, APRA, ABLIS)      (JSON, PDF, PNG evidence)
```

## 7. Key Components & Responsibilities

| Component                             | Responsibility                                                                                |
| ------------------------------------- | --------------------------------------------------------------------------------------------- |
| **HS Workflow / Trigger**             | Detect ticket creation/ABN update; call KYC Service                                           |
| **Public API Aggregator**             | Query ABN Lookup, ASIC, ASX, APRA, ABLIS                                                      |
| **Decision Engine**                   | Determine simplified‑KYC eligibility; set flags                                               |
| **Evidence Collector & File Storage** | Download/convert API responses and webpages to JSON/PDF/PNG; upload to S3; return signed URLs |
| **HubSpot Notes Writer**              | Attach plain‑English decision + structured payload + evidence links                           |
| **(Optional) GPT Summariser**         | Convert JSON outcome into readable paragraph                                                  |

## 8. Non‑Goals / Out of Scope

* UBO identification and Equifax look‑ups
* Biometric checks via Stack Go
* Real‑time batch processing (> 100 tickets/day)
* Multi‑tenant architecture

## 9. Risks & Mitigations

| Risk                                                  | Impact                      | Mitigation                                             |
| ----------------------------------------------------- | --------------------------- | ------------------------------------------------------ |
| Public API downtime                                   | Delays automation           | Retry with back‑off; fall back to manual               |
| Incorrect exemption logic                             | Regulatory breach           | Unit tests + peer review                               |
| Evidence capture fails (DOM change, screenshot error) | Missing audit artefacts     | Monitor capture errors; fallback manual upload warning |
| HubSpot auth token expiry                             | Service stops writing notes | Auto‑refresh tokens; monitoring                        |

## 10. Milestones & Rough Timeline

| Sprint                                           | Duration | Deliverables                                                 |
| ------------------------------------------------ | -------- | ------------------------------------------------------------ |
| **0 – Setup**                                    | 0.5 wk   | AWS account, HS private app, S3 bucket `kyc-evidence`        |
| **1 – Core Lookup, Decision & Evidence Capture** | 1 wk     | API aggregator, decision engine, evidence collector, HS note |
| **2 – (Opt.) GPT Summaries**                     | 0.5 wk   | Text generation & tone tuning                                |
| **3 – UAT & Roll‑out**                           | 0.5 wk   | Test scripts, user training                                  |

## 11. Open Questions

1. Confirm preferred storage location & naming convention for evidence files?
2. Need for region‑locked hosting (AWS Sydney) for data‑sovereignty?
3. Should GPT run locally (open‑source model) or via OpenAI API?

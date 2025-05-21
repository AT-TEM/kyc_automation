# KYC Automation – Phase 1 (AFSL Compliance)

## 1. Problem Statement / Goal
Manual KYC checks for new counterparties are slow, error-prone and expensive.  
We want an automated, low-code workflow that, on HubSpot ticket creation, gathers free public data, decides whether Ultimate Beneficial Owner (UBO) verification can be skipped under AFSL rules, and writes an auditable note (plus evidence files) back to HubSpot—all running locally in Docker.

## 2. Target Users & Context
* **Primary persona**: Compliance / Operations staff with limited technical skills.  
* Works inside **HubSpot** and occasionally opens a Docker container via UI/scripts.  
* Deployment footprint: single Linux VM or laptop; offline except for outbound API calls.

## 3. Success Metrics
* ≥ 80 % of new KYC tickets auto-resolved without manual intervention.
* Turn-around time ≤ 5 minutes per ticket (p95).
* 100 % of automated decisions include an attached evidence file and audit note.
* Zero compliance exceptions in quarterly AFSL audit.

## 4. User Stories
* **As a** compliance officer **I want** a new KYC ticket to trigger an automated lookup **so that** I spend no time collecting public data.  
* **As a** compliance officer **I want** the system to decide if UBO checks can be skipped **so that** we avoid unnecessary Equifax/Stack Go costs.  
* **As a** compliance officer **I want** an English audit note and PDF evidence stored on the ticket **so that** auditors can verify the logic.

## 5. Assumptions & Constraints
* Scope is **Phase 1 only** (no Equifax, email outreach, or Stack Go triggers).
* Runs inside a **single Docker container**; no external orchestrator.
* Only **free public APIs** (ABN Lookup, ASIC, ASX, APRA, ABLIS).
* Internet access may be fire-walled; all endpoints must be allow-listed.
* No PII is stored outside HubSpot.

## 6. Solution Overview
1. **Trigger** – HubSpot workflow posts ticket ID & entity name/ABN to a local webhook.  
2. **Data Enrichment Service** – Python script (Windsurf-driven) queries public APIs in parallel and caches JSON responses.  
3. **Decision Engine** – Simple rules:  
   * ABN inactive → “manual review”.  
   * Listed entity OR AFSL/licence present OR Government body → skip UBO.  
   * Else → “UBO check required”.  
4. **Evidence Builder** – Assemble responses into a single PDF/HTML file.  
5. **Audit Note Writer** – Post decision summary + attach evidence to HubSpot ticket.  
6. **Logging & Metrics** – JSON log + Prometheus exporter (optional).

(Sequence diagram omitted for brevity.)

## 7. Key Components & Responsibilities
| Component | Responsibility |
|-----------|----------------|
| HubSpot Workflow | Detect ticket creation, call local webhook, store results |
| API Adapters (ABN, ASIC, ASX, APRA, ABLIS) | Fetch & normalise entity data |
| Decision Engine | Apply AFSL skip rules |
| Evidence Generator | Merge API payloads into PDF/HTML |
| HubSpot Client | Update ticket, upload file |
| Docker Image & Entrypoint | Package Python app, Windsurf configs, dependencies |
| Config File (.env / YAML) | API keys, timeouts, endpoint URLs |

## 8. Non-Goals / Out of Scope
* Paid API calls (Equifax) or biometric checks (Stack Go).  
* Email outreach to customers.  
* Advanced GPT explanations (placeholder only).

## 9. Risks & Mitigations
| Risk | Mitigation |
|------|------------|
| API downtime / rate-limits | Retries + fallback to manual review |
| Wrong decision logic | Unit tests, UAT with 20 historical tickets |
| HubSpot auth expiry | Store refresh token, alert on failure |
| Docker environment drift | Pin image versions, CI build nightly |

## 10. Milestones & Rough Timeline
| Sprint (2 wks) | Deliverable |
|----------------|------------|
| **0** – Inception | Confirm requirements, finalise PRD |
| **1** – Prototype | Working API adapters, mocked ticket input |
| **2** – MVP | Decision engine, evidence PDF, HubSpot write-back |
| **3** – Hardening | Error handling, logging, CI tests |
| **4** – UAT & Deploy | Pilot with live tickets, doc hand-over |

## 11. Open Questions
1. Where should the evidence file be stored if HubSpot attachment fails?  
2. Preferred format: PDF, HTML, or both?  
3. Do we need Prometheus/Grafana dashboards from day 1?  
4. Minimum Python version & base image preference (e.g., python:3.12-slim)?  

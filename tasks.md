# Task Breakdown: KYC Automation – Phase 1

## Epic 1 – Requirements & Design
- **T-1** Finalise acceptance criteria  
  - *Subtasks*  
    1. ST-1.1 Review PRD with stakeholders  
    2. ST-1.2 Sign-off checklist  
  - *Dependencies*: None  
  - *Deliverable*: signed-off PRD

- **T-2** System architecture diagram  
  - *Subtasks*  
    1. ST-2.1 Draw sequence & component diagrams  
    2. ST-2.2 Add to repo README  
  - *Dependencies*: T-1  
  - *Deliverable*: `docs/architecture.md`

## Epic 2 – HubSpot Integration
- **T-3** Create ticket-triggered workflow  
  - *Subtasks*  
    1. ST-3.1 Configure “KYC Flow” pipeline webhook  
    2. ST-3.2 Test with dummy ticket  
  - *Dependencies*: T-1  
  - *Deliverable*: HubSpot workflow ID

- **T-4** Implement HubSpot API client in Python  
  - *Subtasks*  
    1. ST-4.1 OAuth token refresh helper  
    2. ST-4.2 Wrapper: get ticket, post note, upload file  
  - *Dependencies*: T-3  
  - *Deliverable*: `src/hubspot_client.py`

## Epic 3 – API Adapters
- **T-5** ABN Lookup adapter  
- **T-6** ASIC Professional Registers adapter  
- **T-7** ASX listings scraper/API adapter  
- **T-8** APRA registry adapter  
- **T-9** ABLIS licence adapter  

  *For each T-5 … T-9*  
  - *Subtasks*  
    1. ST-x.1 Draft Pydantic model for response  
    2. ST-x.2 Fetch & parse JSON/HTML  
    3. ST-x.3 Unit tests with fixtures  
  - *Dependencies*: T-1  
  - *Deliverable*: `src/adapters/<name>.py`

## Epic 4 – Decision Engine
- **T-10** Define rule set (YAML)  
  - *Subtasks*: edit `config/rules.yml`  
  - *Dependencies*: T-5-T-9  
  - *Deliverable*: rules file

- **T-11** Implement engine logic  
  - *Subtasks*  
    1. ST-11.1 Load rules YAML  
    2. ST-11.2 Evaluate data & return verdict object  
  - *Dependencies*: T-10  
  - *Deliverable*: `src/decision_engine.py`

## Epic 5 – Evidence Generation
- **T-12** Create HTML template (Jinja2)  
- **T-13** HTML → PDF conversion (WeasyPrint)  

  *Subtasks for each*  
  - Build template, insert API data  
  - Write integration test checking PDF not blank  
  - *Dependencies*: T-5-T-11  
  - *Deliverable*: `templates/evidence.html`, `src/pdf_builder.py`

## Epic 6 – Dockerisation & CI
- **T-14** Write Dockerfile & docker-compose  
- **T-15** GitHub Actions CI: lint, tests, build image  

  *Dependencies*: T-4-T-13  
  - *Deliverable*: `Dockerfile`, `.github/workflows/ci.yaml`

## Epic 7 – Monitoring & Logging
- **T-16** Structured JSON logging (loguru)  
- **T-17** Optional Prometheus exporter  

  *Dependencies*: T-11  
  - *Deliverable*: `src/log_config.py`, `src/metrics.py`

## Epic 8 – UAT & Deployment
- **T-18** Pilot with 20 historical tickets  
  - *Subtasks*: dry-run, compare outcomes  
  - *Dependencies*: Epics 2-7  
  - *Deliverable*: test report

- **T-19** Production rollout  
  - *Subtasks*: docker runbook, hand-over session  
  - *Dependencies*: T-18  
  - *Deliverable*: deployment checklist

---

### Implementation Hints
* Use **httpx + tenacity** for async calls with retries.  
* Cache API responses in SQLite (`utils/cache.py`) for quicker re-runs.  
* Validate rules via **pytest-parametrize** edge-case suite.  
* For PDF, WeasyPrint avoids wkhtmltopdf’s headless-Chrome overhead.  
* Pin Docker base to `python:3.12-slim` and scan with `trivy`.

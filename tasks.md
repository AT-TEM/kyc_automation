# Task Breakdown: KYC Automation (Phase 1 – Local Docker)

> **Repository**: [https://github.com/AT-TEM/kyc\_automation](https://github.com/AT-TEM/kyc_automation)

---

### Legend

* **T‑X** = Task • **ST‑X.Y** = Sub‑task • Dependencies list upstream IDs

---

## Epic 1 – Project Bootstrap & Repository

| ID      | Title                 | Sub‑tasks                                                                                                                                                                                                                                                                                                     | Dependencies | Deliverable                           |
| ------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------------- |
| **T‑1** | Initialise repo ✅     | 1. ST‑1.1 Create GitHub skeleton (README, MIT LICENSE, `.gitignore`, `pre‑commit`) <br>2. ST‑1.2 Add `docs/` folder; commit `prd.md` & `tasks.md`                                                                                                                                                             | —            | Repo scaffold pushed to `main`        |
| **T‑2** | Local dev environment | 1. ST‑2.1 `requirements.txt` with pinned deps (`fastapi`, `uvicorn`, `requests`, `playwright`, `hubspot‑api‑client`, `python‑dotenv`, `pytest`, `ruff`, `black`) <br>2. ST‑2.2 `Makefile` targets: `venv`, `docker-build`, `docker-run`, `test`, `lint` <br>3. ST‑2.3 `.env.example` with placeholder secrets | T‑1          | `make test` runs green on fresh clone |

---

## Epic 2 – Public Data Aggregator

| ID      | Title                    | Sub‑tasks                                                                                                                                                                                                                                                                   | Dependencies | Deliverable                     |
| ------- | ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------------------- |
| **T‑3** | ABN Lookup Client        | 1. ST‑3.1 `lookup_by_abn(abn:str,guid:str)->dict` (retry w/ expo back‑off) <br>2. ST‑3.2 `search_by_name(name:str,guid:str)->list[dict]` returns possible entities <br>3. ST‑3.3 Unit tests w/ pytest‑vcr                                                                   | T‑2          | `src/kyc_service/abn_lookup.py` |
| **T‑4** | Licence & Listing Checks | 1. ST‑4.1 ASIC AFSL scraper/API – given ABN **or** entity name <br>2. ST‑4.2 ASX listing check – daily CSV; cache 24 h inside container volume <br>3. ST‑4.3 APRA / ABLIS licence endpoints <br>4. ST‑4.4 `LicenceStatus` dataclass aggregation <br>5. ST‑4.5 Tests (respx) | T‑3          | `licence_checks.py`             |

---

## Epic 3 – Decision Engine

| ID      | Title                       | Sub‑tasks                                                                                                                                                                                               | Dependencies | Deliverable          |
| ------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | -------------------- |
| **T‑5** | Encode simplified‑KYC rules | 1. ST‑5.1 `Decision` dataclass (`simplified`, `rationale`, `checks`) <br>2. ST‑5.2 `apply_rules()` includes path when ABN absent but licences/listings show legitimacy <br>3. ST‑5.3 Table‑driven tests | T‑3, T‑4     | `decision_engine.py` |

---

## Epic 4 – Evidence Capture & Storage (Local)

| ID      | Title              | Sub‑tasks                                                                                                                                                                                                                                                                                              | Dependencies | Deliverable                                          |
| ------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------ | ---------------------------------------------------- |
| **T‑6** | Evidence Collector | 1. ST‑6.1 Create local mount `evidence/` (Docker volume bind) <br>2. ST‑6.2 `save_abn_json()` writes `abn.json` <br>3. ST‑6.3 `capture_screenshot(url,path)` via Playwright – AFSL & ASX pages <br>4. ST‑6.4 Return filesystem paths; later mapped to HS links <br>5. ST‑6.5 Tests (pytest‑playwright) | T‑3, T‑4     | `evidence_collector.py`; sample files in `evidence/` |

---

## Epic 5 – HubSpot Integration

| ID      | Title                         | Sub‑tasks                                                                                                                                                                           | Dependencies | Deliverable                                    |
| ------- | ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ---------------------------------------------- |
| **T‑7** | HubSpot Note Writer           | 1. ST‑7.1 Upsert custom properties (`is_simplified_kyc`, `kyc_rationale`, `evidence_links`) <br>2. ST‑7.2 Attach timeline note (Markdown) <br>3. ST‑7.3 Handle re‑runs idempotently | T‑5, T‑6     | `hubspot_writer.py`                            |
| **T‑8** | HubSpot Trigger Configuration | 1. ST‑8.1 Create/modify Workflow: pipeline **KYC Flow**, stage **New Customer (AFSL Triggered)** → call webhook <br>2. ST‑8.2 Generate signed webhook secret; store in `.env`       | T‑1          | Workflow live, test ticket hits local endpoint |

---

## Epic 6 – Local Docker Service & CLI

| ID       | Title                       | Sub‑tasks                                                                                                                                                                                                                                                                                                                                                                                                                        | Dependencies | Deliverable                        |
| -------- | --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ---------------------------------- |
| **T‑9**  | FastAPI Webhook Server      | 1. ST‑9.1 `/webhook` POST validates HS signature <br>2. ST‑9.2 Parse ticket JSON; fetch ABN property & ticket name <br>3. ST‑9.3 If ABN **missing**, call `search_by_name`; present CLI selection (TUI via `questionary`) when running in interactive mode; else pick first match in non‑interactive mode <br>4. ST‑9.4 Call orchestrator (lookup → licences → decision → evidence → HS write) <br>5. ST‑9.5 Return 200/4xx JSON | T‑3‑T‑7      | `server.py`                        |
| **T‑10** | Dockerfile & docker‑compose | 1. ST‑10.1 Multi‑stage Dockerfile (slim python → prod) <br>2. ST‑10.2 Expose `PORT` 8000; CMD `uvicorn server:app` <br>3. ST‑10.3 Bind‑mount `./evidence` to `/app/evidence` <br>4. ST‑10.4 `docker-compose.yml` with env‑file `.env`, restart : unless‑stopped                                                                                                                                                                  | T‑2, T‑9     | `Dockerfile`, `docker-compose.yml` |
| **T‑11** | Local Tunnel for Webhooks   | 1. ST‑11.1 Instruction: run `ngrok http 8000` (or Cloudflare tunnel) <br>2. ST‑11.2 Update HS webhook URL with tunnel address (script or manual)                                                                                                                                                                                                                                                                                 | T‑10         | Developer guide section updated    |

---

## Epic 7 – Testing & CI

| ID       | Title                      | Sub‑tasks                                                                                                                                                                                | Dependencies | Deliverable                |
| -------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | -------------------------- |
| **T‑12** | Integration tests (docker) | 1. ST‑12.1 `pytest -m e2e` spins `docker‑compose up -d` <br>2. ST‑12.2 Send sample webhook to `localhost:8000/webhook` <br>3. ST‑12.3 Assert HS mock received note, evidence files exist | T‑10         | `tests/test_e2e.py`        |
| **T‑13** | GitHub Actions CI          | 1. ST‑13.1 Workflow: lint → unit tests → build docker image <br>2. ST‑13.2 Publish to GHCR `ghcr.io/at-tem/kyc_automation` <br>3. ST‑13.3 Optionally push `latest` tag on `main`         | T‑12         | `.github/workflows/ci.yml` |

---

## Epic 8 – Documentation & Roll‑out

| ID       | Title              | Sub‑tasks                                                                                                                                                                               | Dependencies | Deliverable            |
| -------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ---------------------- |
| **T‑14** | Dev & Ops Runbooks | 1. ST‑14.1 README architecture Mermaid diagram <br>2. ST‑14.2 `docs/local_setup.md` – Docker build/run, tunnel setup, env vars <br>3. ST‑14.3 `docs/troubleshooting.md` – common errors | T‑11         | Docs published         |
| **T‑15** | Pilot & Metrics    | 1. ST‑15.1 Run local service for a week <br>2. ST‑15.2 Collect manual vs auto processing times <br>3. ST‑15.3 Review results; plan Phase 2                                              | T‑14         | `docs/pilot_report.md` |

---

## Appendix – Cursor Prompt Snippets (Quick‑copy)

| Component          | Prompt ID   |
| ------------------ | ----------- |
| ABN Lookup         | `prompt‑A`  |
| Name Search        | `prompt‑A2` |
| Licence Checks     | `prompt‑B`  |
| Decision Engine    | `prompt‑C`  |
| Evidence Collector | `prompt‑D`  |
| FastAPI Server     | `prompt‑E`  |
| Dockerfile         | `prompt‑F`  |

*Full templates remain in the earlier Developer Guide; copy‑paste as needed.*

---

### Definition of Done (Phase 1 ‑ Local Docker)

* Docker container builds locally with one command; service reachable at `localhost:8000`.
* HubSpot tickets in stage **New Customer (AFSL Triggered)** auto‑annotated with decision + evidence file paths/links.
* Interactive CLI path resolves tickets w/o ABN.
* ≥ 50 % analyst time‑saving demonstrated in pilot.
* Test coverage ≥ 80 %; CI pipeline green.

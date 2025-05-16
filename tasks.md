# Task Breakdown & Developer Guide – KYC Automation (Phase 1)

> **Repository**: [https://github.com/AT-TEM/kyc\_automation](https://github.com/AT-TEM/kyc_automation)

---

## 0. Repository & Documentation Structure

```
kyc_automation/
├── docs/
│   ├── prd.md              # Product Requirements (already drafted)
│   └── tasks.md            # THIS document – tasks + step‑by‑step guide
├── src/                    # Python packages & entry points
│   ├── kyc_service/
│   │   ├── __init__.py
│   │   ├── abn_lookup.py
│   │   ├── licence_checks.py
│   │   ├── decision_engine.py
│   │   ├── evidence_collector.py
│   │   ├── hubspot_writer.py
│   │   └── lambda_handler.py
│   └── tests/
│       ├── test_abn_lookup.py
│       ├── test_licence_checks.py
│       └── test_decision_engine.py
├── requirements.txt        # Pinned deps (requests, boto3, playwright‑stealth, hubspot‑api‑client,…)
├── .env.example            # Template for secrets
├── deploy_lambda.sh        # One‑click zip + upload script
└── README.md               # Quick‑start, env setup, deploy steps
```

**How to document the PRD**

1. Commit the existing `prd.md` into `docs/` (`git add docs/prd.md`).
2. Commit this `tasks.md` in the same folder.
3. Reference both from `README.md`:

   ```markdown
   ## Project Docs
   * [Product Requirements](docs/prd.md)
   * [Task Breakdown & Guide](docs/tasks.md)
   ```

Push to `main` or a protected branch after PR review.

---

## 1. High‑Level Task List (mirrors PRD milestones)

| ID      | Epic                  | Description                                                             | Deliverable                   |
| ------- | --------------------- | ----------------------------------------------------------------------- | ----------------------------- |
| **T‑1** | Setup                 | AWS account, S3 `kyc-evidence`, IAM roles, HS private app               | AWS README, secrets in `.env` |
| **T‑2** | Public API Aggregator | `abn_lookup.py`, `licence_checks.py` with retry + caching               | Green tests                   |
| **T‑3** | Decision Engine       | `decision_engine.py` + unit tests for exemption logic                   | Coverage ≥ 90 %               |
| **T‑4** | Evidence Collector    | Headless scrape + screenshot + S3 upload utilities                      | PNG/PDF/JSON in bucket        |
| **T‑5** | HubSpot Writer        | Upsert custom props + timeline note with signed URLs                    | Note preview screenshot       |
| **T‑6** | Lambda Wrapper        | `lambda_handler.py` orchestrates T‑2→T‑5; deploy via `deploy_lambda.sh` | Live webhook endpoint         |
| **T‑7** | (Opt.) GPT Summary    | `gpt_summary.py`; add to handler feature flag                           | Readable paragraph in HS      |
| **T‑8** | End‑to‑End Tests      | Mock HS webhook, sample ABNs                                            | Test report                   |
| **T‑9** | Roll‑out & Docs       | README, demo GIF, knowledge‑base article                                | Completed checklist           |

> **Tip:** create a GitHub Project board mapping these task IDs to issues.

---

## 2. Step‑by‑Step Guide

### 2.1 Local Setup

1. **Clone** the repo:

   ```bash
   git clone https://github.com/AT-TEM/kyc_automation.git
   cd kyc_automation
   git checkout -b phase1-automation
   ```
2. **Python env** (3.11 recommended):

   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```
3. **Secrets**: copy `.env.example` ➜ `.env` and fill in:

   ```bash
   HUBSPOT_PRIVATE_APP_TOKEN=...
   ABN_GUID=...
   S3_BUCKET=kyc-evidence
   AWS_REGION=ap-southeast-2
   OPENAI_API_KEY=...   # optional
   ```

### 2.2 AWS & HubSpot

1. **S3 bucket** `kyc-evidence` (Sydney region) with default encryption.
2. **IAM role** `kyc-lambda-role` → policies: `AWSLambdaBasicExecutionRole`, S3 read/write.
3. **Lambda function** `kycService` runtime Python 3.11; set env vars; upload via `deploy_lambda.sh`.
4. **HubSpot Private App** → CRM scope; add **Webhook**:

   * Trigger: *Ticket created* OR *`abn_number` property updated*.
   * URL: Lambda HTTPS endpoint.

### 2.3 Coding with Cursor – Prompt Templates

Copy‑paste these into Cursor to generate starter code.

> **Prompt A – ABN Lookup Client**
>
> ```
> Using `requests`, create a function `lookup_abn(abn: str, guid: str) -> dict` that
> 1) calls the ABR SOAP/JSON service with retries (back‑off 0.5,1,2 s),
> 2) parses active status, entity name, and returns the full JSON.
> Include unit tests with pytest + pytest‑vcr.
> ```

> **Prompt B – Licence & Listing Checks**
>
> ```
> Write async functions to verify if an entity (ABN + name) is:
> • AFSL‑licensed (ASIC Professional Register…)
> • ASX‑listed (download daily CSV; cache for 24 h)
> • Holds APRA or ABLIS licence (public JSON endpoints)
> Return a `LicenceStatus` dataclass.
> Provide mocking tests.
> ```

> **Prompt C – Decision Engine Logic**
>
> ```
> Encode simplified‑KYC rules: skip UBO if any of licence/listing flags true.
> Accept `LicenceStatus`, output `Decision(rationale:str, simplified:bool)`.
> Include table‑driven tests covering all branches.
> ```

> **Prompt D – Evidence Collector**
>
> ```
> Build `EvidenceCollector` that
> • Saves ABN JSON → S3 as `<ticket>/<abn>/abn.json`.
> • Uses Playwright headless to screenshot AFSL or ASX page (full page PNG).
> • Converts HTML to PDF if needed.
> • Returns dict of signed URLs.
> Use moto‑s3 in tests.
> ```

> **Prompt E – HubSpot Writer**
>
> ```
> Implement `HubSpotClient.write_note(ticket_id:int, decision:Decision, files:dict)`.
> Use `hubspot-api-client` SDK.
> ```

> **Prompt F – Lambda Orchestrator**
>
> ```
> Glue components: receive webhook JSON, parse ticket & ABN,
> run lookup → licence checks → decision → evidence → write HS note.
> Return 200/400/500 accordingly.
> Add logging & basic metrics.
> ```

### 2.4 Deployment

1. `bash deploy_lambda.sh` – zips `src/` + deps layer, updates Lambda, deploys.
2. Run test webhook (sample payload in `scripts/sample_ticket.json`).
3. Verify note + evidence URLs appear in test HubSpot ticket.

### 2.5 CI / CD (Optional)

* Configure GitHub Action:

  * steps: `pip install -r requirements.txt`, `pytest`, zip artefact, deploy on `main`.

---

## 3. Coding Conventions & Quality Gates

* **Black + Ruff** enforced via pre‑commit.
* **Pytest** with coverage ≥ 80 %.
* **Type hints** (mypy strict) in `src/`.
* **Retry logic**: use `tenacity` with jitter.
* **Secrets** never committed.

---

## 4. Definition of Done

1. Lambda passes end‑to‑end tests for active & exempt ABNs.
2. HubSpot ticket shows decision note + evidence links.
3. README + docs up to date.
4. Average analyst time reduced to target in pilot run (track manually for 1 week).

---

> **Next Phase Preview**: integrate Equifax + Stack Go once Phase 1 stabilises.

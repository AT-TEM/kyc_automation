# KYC Automation (Phase 1)

A lightweight, Dockerised service that listens to HubSpot ticket webhooks, runs public‑registry checks (ABN, ASIC, ASX, APRA, ABLIS), decides whether simplified KYC applies, captures evidence (files + screenshots), and writes an audit note back to the ticket.

## Quick Start
```bash
# 1. Clone
$ git clone https://github.com/AT-TEM/kyc_automation.git && cd kyc_automation

# 2. Configure environment
$ cp .env.example .env  # then edit secrets

# 3. Build & run
$ docker compose up --build


---

### `docker/Dockerfile`
```Dockerfile
FROM python:3.11-slim
WORKDIR /app

# System deps
RUN apt-get update && apt-get install -y \
    curl jq cron \ \
    && rm -rf /var/lib/apt/lists/*

# Python deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Source
COPY src/ ./src/
COPY docker/entrypoint.sh ./entrypoint.sh
RUN chmod +x ./entrypoint.sh

ENV PYTHONPATH="/app/src"
EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]

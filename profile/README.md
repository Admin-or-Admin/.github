# CyberControl Aurora

**AI-Powered Cybersecurity Control Center**

CyberControl Aurora is an event-driven platform that ingests logs from any source, classifies them using AI, and autonomously investigates and responds to cybersecurity threats — with a human always in control of the final decision.

Every component is an independent service. They communicate exclusively through Apache Kafka topics. Nothing calls anything else directly. This means any service can be restarted, replaced, or scaled without touching the others.

---

## Table of contents

- [How it works](#how-it-works)
- [Repositories](#repositories)
- [Architecture](#architecture)
- [Kafka topics](#kafka-topics)
- [Data flow](#data-flow)
- [Technology stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Service-by-service setup](#service-by-service-setup)
  - [Infrastructure (.github)](#infrastructure-github)
  - [ingestor](#ingestor)
  - [correlator](#correlator)
  - [rac-agents (classifier · analyst · responder)](#rac-agents-classifier--analyst--responder)
  - [ledger](#ledger)
  - [gateway](#gateway)
  - [dashboard](#dashboard)
- [Environment variables reference](#environment-variables-reference)
- [Database schema](#database-schema)
- [Recommended start order](#recommended-start-order)
- [Adding a new log source](#adding-a-new-log-source)
- [Per-repository documentation](#per-repository-documentation)

---

## How it works

A raw log enters the system. Within seconds it has been classified, investigated by an AI analyst with access to your organisation's threat knowledge base, and a full remediation plan has been generated — with exact commands, risk levels, and rollback instructions for each step. Steps that are safe and low-risk execute automatically. Steps that carry risk wait for a human to approve or deny them in the dashboard. The entire trail — from raw log to resolved incident — is stored in PostgreSQL and queryable at any time.

The three AI agents at the heart of the pipeline are called the **RAC agents**: Recognise, Analyse, Classify. They are the classifier, analyst, and responder respectively. All three use RAG (Retrieval-Augmented Generation) — you can drop documents into their knowledge folders (PDFs, DOCX, Markdown) and they will embed, cache, and retrieve the most relevant sections at inference time, making every AI call aware of your specific environment.

---

## Repositories

| Repository | Role | Language |
|---|---|---|
| [`.github`](https://github.com/Admin-or-Admin/.github) | Docker Compose for all infrastructure — Kafka, PostgreSQL, Elasticsearch, Kibana | — |
| [`ingestor`](https://github.com/Admin-or-Admin/ingestor) | Pulls logs from external sources and publishes to `logs.unfiltered` | Python |
| [`correlator`](https://github.com/Admin-or-Admin/correlator) | Aggregates multi-source events and detects attack patterns before classification | Python |
| [`rac-agents`](https://github.com/Admin-or-Admin/rac-agents) | The three AI agents: classifier, analyst, responder | Python |
| [`ledger`](https://github.com/Admin-or-Admin/ledger) | Consumes all Kafka topics and persists everything to PostgreSQL | Python |
| [`gateway`](https://github.com/Admin-or-Admin/gateway) | FastAPI REST service — the only thing that reads from the database and serves the frontend | Python |
| [`dashboard`](https://github.com/Admin-or-Admin/dashboard) | React/TypeScript operator dashboard for monitoring, investigation, and remediation | TypeScript |

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     LOG SOURCES                          │
│         Elasticsearch · Mock Generator · GNS3/Syslog     │
└────────────────────────┬─────────────────────────────────┘
                         │
                    [ ingestor ]
                         │
                  logs.unfiltered
                         │
                   [ correlator ]        ← pattern detection + enrichment
                         │
                  logs.unfiltered        ← enriched events
                         │
                  [ classifier ] ←── classifierKnowledge/  (RAG)
                         │
                  logs.categories
                         │
                    [ analyst ] ←────── analystKnowledge/   (RAG)
                         │
                  logs.solver_plan
                         │
                   [ responder ] ←───── responderKnowledge/ (RAG)
                         │
                  logs.solution
                         │
                    [ ledger ] ←── also consumes ALL other topics
                         │
                    PostgreSQL
                         │
                    [ gateway ]   ← FastAPI · REST · PATCH
                         │
                   [ dashboard ]  ← React · TypeScript · Vite
                         │
                    Human Operator
                    approve / deny
                         │
            (PATCH /remediation/{id} → actions topic → responder)
```

All five Kafka topics also feed into the ledger in parallel. The diagram above shows the main spine for clarity.

---

## Kafka topics

All inter-service communication happens through these topics. No service calls another service directly over HTTP.

| Topic | Producer | Consumers | Content |
|---|---|---|---|
| `logs.unfiltered` | ingestor | classifier, ledger | Raw log documents as received from the source |
| `logs.categories` | classifier | analyst, ledger | Log + classification (category, severity, tags, confidence, isCybersecurity) |
| `logs.solver_plan` | analyst | responder, ledger | Log + classification + threat investigation + proposed remediation steps |
| `logs.solution` | responder | ledger | Full resolution plan with per-step approval status, follow-up actions, post-incident summary |
| `analytics` | classifier, analyst, responder | ledger | Heartbeats and per-event statistics from all three RAC agents |
| `actions` | gateway, responder | responder, ledger | Human approval/denial decisions + ingestor timestamp coordination requests |

Topics are created automatically by the services that produce to them. You do not need to create them manually.

---

## Data flow

A single security event from detection to resolution:

```
T+0.0s   Raw log appears in Elasticsearch
T+0.4s   ingestor polls ES, publishes to logs.unfiltered
T+1.6s   classifier receives log, retrieves RAG context, calls GPT-4.1
           → category: security, severity: critical, confidence: 95%
           → publishes to logs.categories
T+5.4s   analyst receives classification, retrieves RAG context, calls GPT-4.1
           → attackVector, complexity, priority, proposedSteps
           → publishes to logs.solver_plan
T+9.5s   responder receives investigation, retrieves RAG context, calls GPT-4.1
           → resolutionMode, immediateActions (auto + pending), followUpActions
           → publishes to logs.solution
           → ledger writes incident + remediation steps to PostgreSQL
           → gateway serves to dashboard via REST
         Human operator reviews plan in dashboard
         Human approves step 3 via PATCH /remediation/3
           → gateway publishes to actions topic
           → responder executes step
           → ledger updates status + executed_at
         Incident resolved ✓
```

---

## Technology stack

| Layer | Technology |
|---|---|
| Message broker | Apache Kafka 3.4 + Zookeeper |
| Database | PostgreSQL 15 |
| AI agents | Python 3.10+ · LangChain · GPT-4.1 (OpenAI) · Gemini 2.5 Flash (Google) |
| RAG / embeddings | OpenAI `text-embedding-3-small` |
| REST API | FastAPI · psycopg2 · Pydantic |
| Frontend | React 18 · TypeScript · Vite · Recharts · Lucide React |
| Kafka client | `kafka-python-ng` |
| Container infra | Docker · Docker Compose |
| Log source | Elasticsearch 8 · Kibana |
| Monitoring | Redpanda Console |

---

## Prerequisites

Before cloning anything, make sure you have:

- **Docker Desktop** (or Docker Engine + Docker Compose) — for all infrastructure
- **Python 3.10+** — for ingestor, rac-agents, ledger, gateway
- **Node.js 18+** and **npm** — for the dashboard
- **Git**
- At least one of:
  - **OpenAI API key** — required for GPT-4.1 (LLM) and `text-embedding-3-small` (RAG embeddings)
  - **Google Gemini API key** — optional LLM fallback (currently commented out in `llm_router.py`, easily re-enabled)

---

## Quick start

This gets the entire platform running locally in one go.

**1. Clone all repositories**

```bash
git clone https://github.com/Admin-or-Admin/.github
git clone https://github.com/Admin-or-Admin/ingestor
git clone https://github.com/Admin-or-Admin/correlator
git clone https://github.com/Admin-or-Admin/rac-agents
git clone https://github.com/Admin-or-Admin/ledger
git clone https://github.com/Admin-or-Admin/gateway
git clone https://github.com/Admin-or-Admin/dashboard
```

**2. Start infrastructure**

```bash
cd .github
docker compose up -d
```

This starts Kafka, Zookeeper, PostgreSQL, Elasticsearch, Kibana, and Redpanda Console. Wait about 30 seconds for everything to be healthy.

Verify:
- Redpanda Console → http://localhost:8080
- Kibana → http://localhost:5601
- Elasticsearch → http://localhost:9200

**3. Configure environment variables**

Each service reads from a `.env` file. See the [Environment variables reference](#environment-variables-reference) section for all values. At minimum you need your `OPENAI_API_KEY` in `rac-agents/.env`.

```bash
# rac-agents
cp rac-agents/.env rac-agents/.env.backup
# edit rac-agents/.env and set OPENAI_API_KEY

# gateway
echo "DATABASE_URL=postgresql://admin:secret@localhost:5432/cybercontrol" > gateway/.env

# dashboard
echo "VITE_API_URL=http://localhost:8000" > dashboard/.env
echo "VITE_OPENAI_API_KEY=your_openai_key" >> dashboard/.env
```

**4. Set up Python virtual environments**

```bash
# ingestor
cd ingestor && python -m venv venv && source venv/bin/activate
pip install -r requirements.txt && deactivate && cd ..

# rac-agents
cd rac-agents && python -m venv venv && source venv/bin/activate
pip install -r requirements.txt && deactivate && cd ..

# ledger
cd ledger && python -m venv venv && source venv/bin/activate
pip install -r requirements.txt && deactivate && cd ..

# gateway
cd gateway && python -m venv venv && source venv/bin/activate
pip install -r requirements.txt && deactivate && cd ..
```

**5. Install dashboard dependencies**

```bash
cd dashboard && npm install && cd ..
```

**6. Start all services** (each in its own terminal tab)

```bash
# Terminal 1 — ledger
cd ledger && source venv/bin/activate && python main.py

# Terminal 2 — ingestor (mock mode)
cd ingestor && source venv/bin/activate
ENABLED_INGESTORS=mock_logs PYTHONPATH=. python -m ingestor.main

# Terminal 3 — classifier
cd rac-agents && source venv/bin/activate && python classifier.py

# Terminal 4 — analyst
cd rac-agents && source venv/bin/activate && python analyst.py

# Terminal 5 — responder
cd rac-agents && source venv/bin/activate && python responder.py

# Terminal 6 — gateway
cd gateway && source venv/bin/activate && python main.py

# Terminal 7 — dashboard
cd dashboard && npm run dev
```

**7. Open the dashboard**

→ http://localhost:5173

Within a minute you should see security events appearing in the Log Stream page as the mock ingestor generates them and the pipeline classifies and investigates them.

---

## Service-by-service setup

### Infrastructure (.github)

**Repo:** [Admin-or-Admin/.github](https://github.com/Admin-or-Admin/.github)

Contains the Docker Compose file that starts all platform infrastructure. Run it once and leave it running for the lifetime of your development session.

```bash
cd .github
docker compose up -d       # start everything in the background
docker compose ps          # check all containers are healthy
docker compose down        # stop everything (data is preserved in volumes)
docker compose down -v     # stop everything and wipe all data
```

**What it starts:**

| Service | Port | Description |
|---|---|---|
| Zookeeper | 2181 | Kafka coordination |
| Kafka | 29092 | Message broker — use this port in all `.env` files |
| Redpanda Console | 8080 | Kafka UI — browse topics and messages |
| PostgreSQL | 5432 | Primary database — `db: cybercontrol, user: admin, pass: secret` |
| Elasticsearch | 9200 | Log source and search |
| Kibana | 5601 | Elasticsearch UI |

---

### ingestor

**Repo:** [Admin-or-Admin/ingestor](https://github.com/Admin-or-Admin/ingestor)  
**Publishes to:** `logs.unfiltered`

Pulls logs from external sources and streams them into the pipeline. Uses an adapter pattern — each source is one file.

**Available adapters:**

| Adapter | `ENABLED_INGESTORS` value | What it does |
|---|---|---|
| Elasticsearch | `elasticsearch` | Polls an ES index for new documents using a timestamp cursor. Coordinates with ledger on startup to resume from where it left off |
| Mock Generator | `mock_logs` | Generates synthetic security events (SQL injection, XSS, brute force, etc.) and writes them to Elasticsearch. Use this for local development |
| GNS3 | `gns3` | Simulates Cisco network device events + listens on UDP port 1514 for real syslog from GNS3 nodes |

**Setup:**

```bash
cd ingestor
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

**`.env`:**

```properties
ENABLED_INGESTORS=elasticsearch,mock_logs
ELASTIC_HOST=http://localhost:9200
ELASTIC_INDEX=mock-logs
KAFKA_BROKERS=localhost:29092
KAFKA_TOPIC=logs.unfiltered
POLL_INTERVAL=1.0
MOCK_DELAY=0.5
SYSLOG_PORT=1514
SIM_INTERVAL=1.0
```

**Run:**

```bash
# Must be run as a module from the parent directory of ingestor/
PYTHONPATH=. python -m ingestor.main

# Run only mock logs (fastest for local dev)
ENABLED_INGESTORS=mock_logs PYTHONPATH=. python -m ingestor.main
```

> **Note on Elasticsearch client and TLS:** the ES Python client v8+ defaults to HTTPS. If you see SSL errors against the local Docker ES instance, ensure `ELASTIC_HOST` starts with `http://`, not `https://`.

---

### correlator

**Repo:** [Admin-or-Admin/correlator](https://github.com/Admin-or-Admin/correlator)  
**Consumes:** `logs.unfiltered`  
**Publishes:** enriched events back to `logs.unfiltered`

Sits between the ingestor and the classifier. Aggregates events from multiple sources, detects attack patterns across events (e.g. a port scan followed by a failed login from the same IP), and enriches each event with correlation metadata before it reaches the classifier.

See the repository README for setup instructions specific to the correlator.

---

### rac-agents (classifier · analyst · responder)

**Repo:** [Admin-or-Admin/rac-agents](https://github.com/Admin-or-Admin/rac-agents)

Three agents, one repository. Each is a standalone Python process. RAC stands for **Recognise · Analyse · Counter**.

**Setup:**

```bash
cd rac-agents
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
mkdir -p classifierKnowledge analystKnowledge responderKnowledge
```

**`.env`:**

```properties
# LLM keys — at least OPENAI_API_KEY is required
OPENAI_API_KEY=sk-...
GEMINI_API_KEY_1=AIza...        # optional, uncomment in llm_router.py to use
GEMINI_API_KEY_2=AIza...        # optional second Gemini key for rotation

# Kafka
KAFKA_BROKERS=localhost:29092
KAFKA_GROUP_ID=classifier-group

# Agent modes: 'kafka' for production, 'manual' for testing single logs
CLASSIFIER_MODE=kafka
ANALYST_MODE=kafka
RESPONDER_MODE=kafka

# Confidence threshold — analyst skips logs below this %
# Set to 0 to pass everything through
MIN_CLASSIFICATION_CONFIDENCE=0

# LLM rate limit behaviour
RATE_LIMIT_WAIT_SECONDS=60

# Heartbeat intervals in seconds
CLASSIFIER_HEARTBEAT_INTERVAL=30
ANALYST_HEARTBEAT_INTERVAL=30
RESPONDER_HEARTBEAT_INTERVAL=30

# RAG — classifier
KNOWLEDGE_DIR=classifierKnowledge
KNOWLEDGE_CHUNK_SIZE=500
KNOWLEDGE_CHUNK_OVERLAP=50
KNOWLEDGE_TOP_K=5

# RAG — analyst
ANALYST_KNOWLEDGE_DIR=analystKnowledge
ANALYST_KNOWLEDGE_CHUNK_SIZE=500
ANALYST_KNOWLEDGE_CHUNK_OVERLAP=50
ANALYST_KNOWLEDGE_TOP_K=5

# RAG — responder
RESPONDER_KNOWLEDGE_DIR=responderKnowledge
RESPONDER_KNOWLEDGE_CHUNK_SIZE=500
RESPONDER_KNOWLEDGE_CHUNK_OVERLAP=50
RESPONDER_KNOWLEDGE_TOP_K=5
```

**Run (each in its own terminal):**

```bash
cd rac-agents && source venv/bin/activate

python classifier.py    # listens on logs.unfiltered
python analyst.py       # listens on logs.categories
python responder.py     # listens on logs.solver_plan
```

**LLM rotation:** the router tries keys in order — Gemini key 1 → Gemini key 2 → GPT-4.1. On a 429 it rotates automatically. If all keys are exhausted it waits `RATE_LIMIT_WAIT_SECONDS` and resets. Currently only GPT-4.1 is active; Gemini keys are present in `.env` but commented out in `llm_router.py`. To enable them, uncomment the Gemini client blocks in `llm_router.py`.

**RAG knowledge bases:**

Drop any `.pdf`, `.docx`, `.txt`, or `.md` file into the relevant folder and restart the agent. It will embed on first run and cache the result — subsequent restarts load from cache instantly if nothing has changed.

| Agent | Folder | What to put here |
|---|---|---|
| classifier | `classifierKnowledge/` | Internal log format docs, service catalogue, categorisation rules, compliance requirements |
| analyst | `analystKnowledge/` | Threat intelligence, CVE databases, MITRE ATT&CK descriptions, past incident notes |
| responder | `responderKnowledge/` | Remediation playbooks, approved command lists, runbooks, escalation policies |

**Manual mode (testing without Kafka):**

Set `CLASSIFIER_MODE=manual`, `ANALYST_MODE=manual`, `RESPONDER_MODE=manual` in `.env`, then run each agent one at a time. The classifier reads from stdin; each subsequent agent reads from `pipeline.json` which the previous one wrote.

---

### ledger

**Repo:** [Admin-or-Admin/ledger](https://github.com/Admin-or-Admin/ledger)  
**Consumes:** all six Kafka topics  
**Writes to:** PostgreSQL

The only service that writes to the database. Subscribes to `logs.unfiltered`, `logs.categories`, `logs.solver_plan`, `logs.solution`, `analytics`, and `actions` and progressively enriches database rows as each log moves through the pipeline.

Also handles the ingestor's startup timestamp request — it responds to `get_last_unfiltered_timestamp` messages on the `actions` topic so the ingestor knows where to resume polling after a restart.

**Setup:**

```bash
cd ledger
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

The database and all tables are created automatically on first startup. You only need to ensure the PostgreSQL container is running and the `cybercontrol` database exists:

```bash
# If using the Docker Compose setup this is already done.
# If connecting to a separate PostgreSQL instance:
psql -U admin -h localhost -c "CREATE DATABASE cybercontrol;"
```

**`.env`:**

```properties
KAFKA_BROKERS=localhost:29092
KAFKA_GROUP_ID=ledger-group
DATABASE_URL=postgresql://admin:secret@localhost:5432/cybercontrol
```

**Run:**

```bash
cd ledger && source venv/bin/activate && python main.py
```

---

### gateway

**Repo:** [Admin-or-Admin/gateway](https://github.com/Admin-or-Admin/gateway)  
**Reads from:** PostgreSQL  
**Serves:** REST API on port 8000

The FastAPI service that sits between the database and the dashboard. The only service that reads from PostgreSQL externally. All dashboard data comes through here.

**Setup:**

```bash
cd gateway
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

**`.env`:**

```properties
DATABASE_URL=postgresql://admin:secret@localhost:5432/cybercontrol
PORT=8000
HOST=0.0.0.0
```

**Run:**

```bash
cd gateway && source venv/bin/activate && python main.py
```

The API is available at http://localhost:8000. Interactive docs at http://localhost:8000/docs.

**Full API reference:**

| Method | Endpoint | Description |
|---|---|---|
| GET | `/logs` | Paginated log list. Optional `?service_name=` filter |
| GET | `/logs/{id}` | Single log |
| GET | `/logs/{id}/details` | Log + classification + threat assessment |
| GET | `/logs/{id}/full` | Log + classification + threat + incident + remediation |
| GET | `/classifications` | All classifications |
| GET | `/classifications/{log_id}` | Classification for a specific log |
| GET | `/threats` | All threat assessments |
| GET | `/threats/{log_id}` | Threat assessment for a specific log |
| GET | `/incidents` | All incidents |
| GET | `/incidents/{id}` | Single incident |
| GET | `/incidents/{id}/remediation` | Remediation steps for an incident |
| GET | `/incidents/{id}/actions` | Follow-up actions for an incident |
| GET | `/remediation` | All remediation steps |
| GET | `/remediation/log/{log_id}` | Remediation steps for a specific log |
| PATCH | `/remediation/{step_id}` | Update step status — body: `{"status": "approved"}` or `{"status": "denied"}` |
| GET | `/follow-ups` | All follow-up actions |
| GET | `/follow-ups/incident/{id}` | Follow-up actions for a specific incident |
| GET | `/stats/severity` | Severity distribution counts |
| GET | `/stats/incidents/trend` | Incident count over time |
| GET | `/stats/services` | Per-service log counts |

---

### dashboard

**Repo:** [Admin-or-Admin/dashboard](https://github.com/Admin-or-Admin/dashboard)  
**Connects to:** gateway at `VITE_API_URL`  
**Runs on:** http://localhost:5173

The operator-facing React application. Six pages.

**Setup:**

```bash
cd dashboard
npm install
```

**`.env`:**

```properties
VITE_API_URL=http://localhost:8000
VITE_OPENAI_API_KEY=sk-...
```

**Run:**

```bash
cd dashboard && npm run dev
```

**Pages:**

| Page | Route | Description |
|---|---|---|
| Overview | `/` | KPI tiles, severity chart, incident trend, services breakdown |
| Log Stream | `/logs` | Live feed of `logs.unfiltered` events, auto-refreshes every 5s |
| Threats | `/threats` | Threat assessments table with detail panel and Ask AI button |
| Incidents | `/incidents` | Incidents table with drill-down and Ask AI button |
| Remediation | `/remediation` | Tabbed remediation steps — approve or deny each step |
| Agent Monitor | `/agents` | Derived agent health metrics from the analytics topic |
| AI Analyst | `/analyst` | Agentic chat interface powered by GPT-4.1 with 15 tools mapped to all gateway endpoints |

**AI Analyst:** the chat interface runs a GPT-4.1 agentic loop (up to 10 iterations per turn) with tools for every gateway endpoint. It can be opened directly from the Threats and Incidents pages via the **Ask AI** button, which pre-populates an investigation query for the selected event.

---

## Environment variables reference

Complete reference for every variable across all services.

### Infrastructure

| Variable | Value | Description |
|---|---|---|
| PostgreSQL | `postgresql://admin:secret@localhost:5432/cybercontrol` | Default connection used by ledger and gateway |
| Kafka | `localhost:29092` | Default broker address for all services |

### ingestor

| Variable | Default | Description |
|---|---|---|
| `ENABLED_INGESTORS` | `elasticsearch,mock_logs` | Comma-separated list of adapters to run |
| `ELASTIC_HOST` | `http://localhost:9200` | Elasticsearch URL including scheme |
| `ELASTIC_INDEX` | `mock-logs` | Index to read from / write to |
| `KAFKA_BROKERS` | `localhost:29092` | Kafka broker address |
| `KAFKA_TOPIC` | `logs.unfiltered` | Topic to publish to |
| `POLL_INTERVAL` | `1.0` | Seconds between ES polls when caught up |
| `MOCK_DELAY` | `0.5` | Base delay between mock event generation |
| `SYSLOG_PORT` | `1514` | UDP port for GNS3 syslog listener |
| `SIM_INTERVAL` | `1.0` | Seconds between simulated GNS3 events |

### rac-agents

| Variable | Default | Description |
|---|---|---|
| `OPENAI_API_KEY` | required | OpenAI key — GPT-4.1 (LLM) + text-embedding-3-small (RAG) |
| `GEMINI_API_KEY_1` | optional | First Gemini key (uncomment in llm_router.py to activate) |
| `GEMINI_API_KEY_2` | optional | Second Gemini key for rotation |
| `KAFKA_BROKERS` | `localhost:29092` | Kafka broker address |
| `KAFKA_GROUP_ID` | `classifier-group` | Consumer group — change only to force full replay |
| `CLASSIFIER_MODE` | `manual` | `kafka` for production, `manual` for stdin testing |
| `ANALYST_MODE` | `manual` | `kafka` or `manual` |
| `RESPONDER_MODE` | `manual` | `kafka` or `manual` |
| `MIN_CLASSIFICATION_CONFIDENCE` | `70` | Analyst skips logs below this %. Set to `0` to disable |
| `RATE_LIMIT_WAIT_SECONDS` | `60` | Wait time when all LLM keys are rate limited |
| `CLASSIFIER_HEARTBEAT_INTERVAL` | `30` | Seconds between classifier heartbeats |
| `ANALYST_HEARTBEAT_INTERVAL` | `30` | Seconds between analyst heartbeats |
| `RESPONDER_HEARTBEAT_INTERVAL` | `30` | Seconds between responder heartbeats |
| `KNOWLEDGE_DIR` | `classifierKnowledge` | Classifier RAG folder |
| `KNOWLEDGE_CHUNK_SIZE` | `500` | Words per chunk |
| `KNOWLEDGE_CHUNK_OVERLAP` | `50` | Overlap between chunks |
| `KNOWLEDGE_TOP_K` | `5` | Chunks retrieved per call |
| `ANALYST_KNOWLEDGE_DIR` | `analystKnowledge` | Analyst RAG folder |
| `ANALYST_KNOWLEDGE_CHUNK_SIZE` | `500` | |
| `ANALYST_KNOWLEDGE_CHUNK_OVERLAP` | `50` | |
| `ANALYST_KNOWLEDGE_TOP_K` | `5` | |
| `RESPONDER_KNOWLEDGE_DIR` | `responderKnowledge` | Responder RAG folder |
| `RESPONDER_KNOWLEDGE_CHUNK_SIZE` | `500` | |
| `RESPONDER_KNOWLEDGE_CHUNK_OVERLAP` | `50` | |
| `RESPONDER_KNOWLEDGE_TOP_K` | `5` | |

### ledger

| Variable | Default | Description |
|---|---|---|
| `KAFKA_BROKERS` | `localhost:29092` | Kafka broker address |
| `KAFKA_GROUP_ID` | `ledger-group` | Consumer group — change only to force full replay |
| `DATABASE_URL` | `postgresql://admin:secret@localhost:5432/cybercontrol` | PostgreSQL connection string |

### gateway

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql://admin:secret@localhost:5432/cybercontrol` | PostgreSQL connection string |
| `PORT` | `8000` | Port to listen on |
| `HOST` | `0.0.0.0` | Bind address |

### dashboard

| Variable | Default | Description |
|---|---|---|
| `VITE_API_URL` | `http://localhost:8000` | Gateway base URL |
| `VITE_OPENAI_API_KEY` | required | OpenAI key for the AI Analyst chat in the browser |

---

## Database schema

All tables are created by the ledger on startup.

```sql
-- Core log record, progressively enriched through the pipeline
CREATE TABLE logs (
    id               VARCHAR PRIMARY KEY,     -- trace.id from the log event
    timestamp        TIMESTAMPTZ,
    message          TEXT,
    service_name     VARCHAR,
    raw_severity     VARCHAR,
    user_id          VARCHAR,
    http_status      INTEGER,
    process_pid      INTEGER,
    trace_id         VARCHAR,
    processing_stage VARCHAR,                 -- unfiltered → classified → investigated → resolved
    created_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Written by ledger when it receives from logs.categories
CREATE TABLE classifications (
    log_id           VARCHAR PRIMARY KEY REFERENCES logs(id),
    category         VARCHAR,                 -- security / infrastructure / application / deployment
    severity         VARCHAR,                 -- critical / high / medium / low / info
    is_cybersecurity BOOLEAN,
    confidence       INTEGER,                 -- 0–100
    reasoning        TEXT,
    tags             TEXT[],
    created_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Written by ledger when it receives from logs.solver_plan
CREATE TABLE threat_assessments (
    log_id            VARCHAR PRIMARY KEY REFERENCES logs(id),
    attack_vector     TEXT,
    ai_suggestion     TEXT,
    complexity        VARCHAR,                -- simple / complex
    recurrence_rate   INTEGER,               -- 0–100 probability within 24h
    confidence        INTEGER,
    auto_fixable      BOOLEAN,
    requires_approval BOOLEAN,
    priority          INTEGER,               -- 1–5
    notify_teams      TEXT[],
    created_at        TIMESTAMPTZ DEFAULT NOW()
);

-- Written by ledger when it receives from logs.solution
CREATE TABLE incidents (
    incident_id       VARCHAR PRIMARY KEY,
    log_id            VARCHAR REFERENCES logs(id),
    resolution_mode   VARCHAR,               -- autonomous / guided
    executive_summary TEXT,
    outcome           VARCHAR,
    resolved_by       VARCHAR,
    what_happened     TEXT,
    impact_assessment TEXT,
    root_cause        TEXT,
    lessons_learned   TEXT,
    resolved_at       TIMESTAMPTZ,
    created_at        TIMESTAMPTZ DEFAULT NOW()
);

-- One row per step in resolution.immediateActions
CREATE TABLE remediation_steps (
    id                SERIAL PRIMARY KEY,
    log_id            VARCHAR REFERENCES logs(id),
    step_number       INTEGER,
    title             VARCHAR,
    description       TEXT,
    command           TEXT,
    risk              VARCHAR,               -- low / medium / high
    estimated_time    VARCHAR,
    rollback          TEXT,
    auto_execute      BOOLEAN,
    requires_approval BOOLEAN,
    status            VARCHAR,               -- auto / pending / approved / denied
    executed_at       TIMESTAMPTZ,
    created_at        TIMESTAMPTZ DEFAULT NOW()
);

-- One row per item in resolution.followUpActions
CREATE TABLE follow_up_actions (
    id           SERIAL PRIMARY KEY,
    incident_id  VARCHAR REFERENCES incidents(incident_id),
    title        VARCHAR,
    description  TEXT,
    owner        VARCHAR,
    deadline     VARCHAR,                    -- immediate / 24h / 48h / 1 week
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Raw analytics events from all three RAC agents
CREATE TABLE analytics (
    id         SERIAL PRIMARY KEY,
    agent      VARCHAR,                      -- classifier / analyst / responder
    event      VARCHAR,                      -- heartbeat / classification_produced / etc.
    payload    JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

A single event can be traced through every table using its `trace.id` which becomes `logs.id`. The `processing_stage` on the `logs` row tells you how far through the pipeline it has progressed at any point in time.

---

## Recommended start order

Start services in this order to ensure each one has what it depends on before coming up.

```
1.  docker compose up -d          (.github — Kafka, PostgreSQL, Elasticsearch)

    Wait ~30 seconds for containers to be healthy.

2.  python main.py                (ledger — must be up before ingestor so it can
                                   respond to the timestamp coordination request)

3.  PYTHONPATH=. python -m ingestor.main   (ingestor — starts publishing to Kafka)

4.  python classifier.py          (rac-agents — begins consuming logs.unfiltered)

5.  python analyst.py             (rac-agents — begins consuming logs.categories)

6.  python responder.py           (rac-agents — begins consuming logs.solver_plan)

7.  python main.py                (gateway — begins serving the API)

8.  npm run dev                   (dashboard — opens at http://localhost:5173)
```

Services 4–8 can technically start in any order since Kafka buffers messages, but the order above produces the cleanest startup logs. The ingestor (step 3) will wait up to 10 seconds for a ledger response on startup and fall back to epoch if the ledger is not yet ready — so the ledger should ideally be up first.

---

## Adding a new log source

The ingestor uses an adapter pattern. To add a new source:

**1. Create `ingestor/adapters/your_source_adapter.py`:**

```python
from .base import BaseAdapter
from shared.kafka_client import AuroraProducer
import os, time

class YourSourceAdapter(BaseAdapter):
    def __init__(self, name="YourSource"):
        super().__init__(name)
        self.kafka_brokers = [os.getenv("KAFKA_BROKERS", "localhost:29092")]
        self.kafka_topic   = os.getenv("KAFKA_TOPIC", "logs.unfiltered")

    def run(self):
        producer = AuroraProducer(self.kafka_brokers)
        producer.ensure_topic(self.kafka_topic)

        while True:
            for event in fetch_from_your_source():
                producer.send_log(self.kafka_topic, {
                    "@timestamp":  event["time"],
                    "message":     event["message"],
                    "service.name": event["service"],
                    "trace.id":    event["id"],
                    "log.level":   event["level"],
                })
            producer.flush()
            time.sleep(5)
```

**2. Register it in `ingestor/main.py`:**

```python
from ingestor.adapters.your_source_adapter import YourSourceAdapter

ADAPTER_MAP = {
    "elasticsearch": ElasticsearchAdapter,
    "gns3":          GNS3Adapter,
    "mock_logs":     MockLogsAdapter,
    "your_source":   YourSourceAdapter,   # add this line
}
```

**3. Enable it:**

```properties
ENABLED_INGESTORS=your_source
```

The minimum fields every event must include are `@timestamp`, `message`, `service.name`, and `trace.id`. All other fields are optional. Nothing else in the pipeline needs to change.

---

## Per-repository documentation

Each repository has its own detailed README covering setup, configuration, internals, and extension points.

| Repository | README covers |
|---|---|
| [`ingestor`](https://github.com/Admin-or-Admin/ingestor) | All three adapters in detail, startup coordination with ledger, log event format, adding new adapters |
| [`rac-agents`](https://github.com/Admin-or-Admin/rac-agents) | Classifier/analyst/responder internals, LLM router, RAG knowledge bases, analytics events, manual mode |
| [`ledger`](https://github.com/Admin-or-Admin/ledger) | Per-topic persistence logic, timestamp coordination, full database schema, table relationships |
| [`gateway`](https://github.com/Admin-or-Admin/gateway) | All endpoints with request/response shapes, database queries, CORS configuration |
| [`dashboard`](https://github.com/Admin-or-Admin/dashboard) | All six pages, AI Analyst tool definitions, Ask AI flow, Remediation approve/deny, api.ts reference |
| [`correlator`](https://github.com/Admin-or-Admin/correlator) | Pattern detection logic, enrichment fields, configuration |

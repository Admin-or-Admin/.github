# Aurora

![OS: Linux](https://img.shields.io/badge/OS-Linux-blue?logo=linux&logoColor=white)
![OS: macOS](https://img.shields.io/badge/OS-macOS-black?logo=apple&logoColor=white)
![OS: Windows](https://img.shields.io/badge/OS-Windows-blue?logo=windows&logoColor=white)

**AI-Powered Autonomous Security Operations Center**

Aurora is an event-driven platform that ingests logs from any observability source, classifies them using AI agents, and autonomously investigates and responds to cybersecurity threats — with a human always in control of the final decision.

Built on Apache Kafka as its central nervous system, Aurora is composed of seven independent microservices that communicate exclusively through Kafka topics, making the architecture fully decoupled and horizontally scalable.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Services](#services)
- [Data Flow](#data-flow)
- [Kafka Topics](#kafka-topics)
- [Database Schema](#database-schema)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Configuration Reference](#configuration-reference)
- [Running Services Individually](#running-services-individually)
- [Manual Mode (No Kafka)](#manual-mode-no-kafka)
- [Extending Aurora](#extending-aurora)
- [Project Structure](#project-structure)
- [Related Repositories](#related-repositories)

---

## Overview

Modern enterprises generate terabytes of log data daily. Traditional SIEM systems lead to alert fatigue — critical signals buried under thousands of false positives. Aurora addresses this by automating Tier-1 and Tier-2 SOC operations through a pipeline of AI agents that classify, correlate, investigate, and remediate security events in real time.

A typical security event travels from detection to a human-ready remediation plan in under 10 seconds.

**Key capabilities:**

- Real-time AI classification of raw logs by severity, category, and security relevance
- Cross-domain threat investigation — the analyst considers infrastructure, deployment, and application logs alongside security events to surface correlations that single-domain analysis misses
- Structured, step-by-step remediation plans with risk assessments and rollback instructions
- Human-in-the-loop governance — high-risk steps require explicit operator approval before any action is taken
- RAG-augmented agents — drop your playbooks, runbooks, and threat intel into knowledge folders and the agents use them automatically
- Complete audit trail — every classification decision, investigation, and remediation step is persisted

---

## Architecture

Aurora uses an **Event-Driven Microservices Architecture**. No service calls another service directly. All inter-service communication flows through Kafka topics. The Ledger is the only service that writes to PostgreSQL — all other services either publish to Kafka or read from the database via the Gateway.

```mermaid
flowchart TD
    subgraph Sources["External Sources"]
        ES[Elasticsearch]
        MOCK[Mock Generator]
        GNS3[GNS3 / Syslog]
    end

    subgraph Ingestion["Ingestion Layer"]
        ING[Ingestor]
    end

    subgraph Kafka["Kafka Topics"]
        LU[logs.unfiltered]
        LC[logs.categories]
        LSP[logs.solver_plan]
        LS[logs.solution]
        AN[analytics]
        ACT[actions]
    end

    subgraph Agents["AI Agent Layer — rac-agents"]
        CLASS[Classifier]
        ANALYST[Analyst]
        RESP[Responder]
    end

    subgraph Persistence["Persistence Layer"]
        LEDGER[Ledger]
        DB[(PostgreSQL)]
    end

    subgraph API["Presentation Layer"]
        GW[Gateway]
        UI[Dashboard]
    end

    ES & MOCK & GNS3 --> ING
    ING --> LU
    LU --> CLASS
    CLASS --> LC
    CLASS --> AN
    LC --> ANALYST
    ANALYST --> LSP
    ANALYST --> AN
    LSP --> RESP
    RESP --> LS
    RESP --> AN

    LU & LC & LSP & LS & AN & ACT --> LEDGER
    LEDGER --> DB
    DB --> GW
    GW --> UI
    UI -->|PATCH /remediation| GW
    GW --> ACT
```

---

## Services

### Ingestor

The entry point of the pipeline. Pulls log events from external sources and publishes them to `logs.unfiltered`. Source-agnostic — each log source is an independent adapter running in its own thread. Adding a new source requires only a new adapter file.

**Adapters:**

| Adapter | Description |
|---|---|
| `elasticsearch` | Polls an Elasticsearch index using a cursor to avoid reprocessing on restart |
| `mock_logs` | Generates synthetic security events and indexes them into Elasticsearch for end-to-end testing |
| `gns3` | Runs a UDP syslog listener on port 1514 and a Cisco-style log simulator for network lab environments |

The Elasticsearch adapter performs a startup handshake with the Ledger over the `actions` topic to retrieve the last processed timestamp. If the Ledger is not yet available, the adapter waits up to 10 seconds before falling back to epoch.

---

### RAC Agents (Classifier, Analyst, Responder)

Three autonomous Python agents forming the core intelligence pipeline. Each is a standalone process that can run on separate machines, scale independently, and restart without affecting the others. All three use an LLM router and can rotate across multiple API keys automatically on rate limit errors.

#### Classifier

Reads raw logs from `logs.unfiltered` and produces a structured classification for each one.

**Output per log:**

| Field | Description |
|---|---|
| `category` | `security`, `infrastructure`, `application`, or `deployment` |
| `severity` | `critical`, `high`, `medium`, `low`, or `info` |
| `tags` | 2–5 descriptive lowercase tags |
| `isCybersecurity` | Whether the log has security relevance |
| `sendToInvestigationAgent` | True only if `isCybersecurity` is true and severity is not `info` |
| `classificationConfidence` | 0–100 model confidence score |
| `reasoning` | One sentence explaining the classification |

#### Analyst

Consumes classified logs from `logs.categories`. Applies two filters before spending tokens: drops logs below `MIN_CLASSIFICATION_CONFIDENCE` and drops non-security events. For logs that pass, it performs a deep threat investigation using the log, its classification, and relevant chunks from `analystKnowledge/`.

**Output per investigation:**

| Field | Description |
|---|---|
| `aiSuggestion` | 2–3 sentences describing the threat |
| `attackVector` | Specific technique or threat pattern identified |
| `complexity` | `simple` or `complex` — drives the responder's resolution mode |
| `autoFixable` | Whether automation can safely resolve this |
| `requiresHumanApproval` | Whether operator sign-off is required |
| `priority` | 1–5 urgency scale |
| `proposedSteps` | Ordered remediation steps with commands, risk levels, and rollback instructions |

#### Responder

Takes the analyst's investigation and produces a final, executable resolution plan that is persisted by the Ledger and surfaced in the Dashboard.

**Resolution modes:**

- `autonomous` — assigned when `autoFixable=true` and `complexity=simple`. Steps tagged `autoExecute: true` are cleared for execution without human approval.
- `guided` — assigned otherwise. Steps tagged `requiresApproval: true` wait for an operator decision in the dashboard before proceeding.

The responder also produces a post-incident summary (`whatHappened`, `rootCause`, `impactAssessment`, `lessonsLearned`) and a set of follow-up actions with owners and deadlines.

#### LLM Router

All three agents call the LLM through `invoke_with_rotation()`. On a 429 rate limit, the router rotates to the next key silently. If all keys are exhausted, it waits `RATE_LIMIT_WAIT_SECONDS` (default 60s) then resets to the first key.

#### RAG Knowledge Bases

Each agent has its own knowledge folder. Drop `.pdf`, `.docx`, `.txt`, or `.md` files into the folder and restart the agent. On startup, the agent chunks, embeds, and caches the content. At inference time, the top-K most relevant chunks are retrieved via cosine similarity and injected into the prompt.

| Agent | Folder | Recommended content |
|---|---|---|
| Classifier | `classifierKnowledge/` | Log format docs, field definitions, service catalogue, custom tagging rules |
| Analyst | `analystKnowledge/` | Threat intel reports, CVE databases, MITRE ATT&CK, past incident notes |
| Responder | `responderKnowledge/` | Remediation playbooks, runbooks, approved command templates, escalation policies |

If a folder does not exist, the agent logs a warning and continues without RAG — it does not crash.

---

### Ledger

The persistence layer. Subscribes to all Kafka topics and upserts each message into PostgreSQL as logs progress through pipeline stages. It is the only service that writes to the database. The Ledger also handles the ingestor's startup timestamp coordination request on the `actions` topic.

---

### Gateway

A REST API that sits between PostgreSQL and the Dashboard. Read-only except for one write endpoint: `PATCH /remediation/{step_id}`, which records human approval or denial decisions and publishes them to the `actions` topic.

The gateway does not connect to Kafka. It is a pure read layer on top of the database.

---

### Dashboard

A React + TypeScript single-page application providing real-time monitoring and human-in-the-loop controls. Connects to the Gateway via REST on load.

**Pages:**

| Page | Description |
|---|---|
| Overview | KPI cards, severity distribution chart, incident trend (30 days), top services by log volume |
| AI Analyst | Agentic chat interface with GPT-4.1. Has access to 15 tools mapped to every gateway endpoint — queries live data, not memory |
| Log Stream | Live feed of `logs.unfiltered` from the database, auto-refreshing every 5 seconds |
| Threats | Table of threat assessments from the Analyst agent with detail panel and Ask AI shortcut |
| Incidents | Incidents table with full drill-down — post-incident summary, all remediation steps, follow-up actions |
| Remediation | All remediation steps across all incidents with Approve / Deny controls for pending steps |
| Agent Monitor | Derived health metrics for all three agents — processing counts, confidence averages, output breakdowns |

---

## Data Flow

A single security event flows through the full pipeline in under 10 seconds from detection to a human-ready remediation plan.

```
14:32:11.000  Event appears in Elasticsearch index
14:32:11.423  → logs.unfiltered     Raw document ingested and published by ingestor
14:32:13.891  → logs.categories    Classified as security / critical (2.4s)
14:32:19.234  → logs.solver_plan   Investigated, remediation plan with 4 steps ready (5.3s)
14:32:19.500  → logs.solution      Responder publishes resolution record
14:34:01.882  → actions            Operator approves steps via dashboard
```

```mermaid
sequenceDiagram
    participant Ingestor
    participant Kafka
    participant Classifier
    participant Analyst
    participant Responder
    participant Ledger
    participant DB as PostgreSQL
    participant GW as Gateway
    participant UI as Dashboard

    Ingestor->>Kafka: publish to logs.unfiltered
    Kafka->>Classifier: consume raw log
    Classifier->>Kafka: publish to logs.categories
    Kafka->>Analyst: consume classified log
    Analyst->>Kafka: publish to logs.solver_plan
    Kafka->>Responder: consume investigation
    Responder->>Kafka: publish to logs.solution
    Kafka->>Ledger: consume all topics
    Ledger->>DB: upsert log row + related tables
    UI->>GW: GET /threats, /incidents, etc.
    GW->>DB: query
    DB-->>GW: rows
    GW-->>UI: JSON response
    UI->>GW: PATCH /remediation/{id} (human approval)
    GW->>Kafka: publish to actions
```

---

## Kafka Topics

All inter-service communication happens exclusively through these topics. No service calls another service directly.

| Topic | Producer | Consumers | Content |
|---|---|---|---|
| `logs.unfiltered` | Ingestor | Classifier, Ledger | Raw log documents as received from the source system |
| `logs.categories` | Classifier | Analyst, Ledger | Logs enriched with `category`, `severity`, `tags`, and `isCybersecurity` |
| `logs.solver_plan` | Analyst | Responder, Ledger | Security logs with full threat investigation and proposed remediation steps |
| `logs.solution` | Responder | Ledger | Complete resolution records including immediate actions and post-incident summary |
| `analytics` | Classifier, Analyst, Responder | Ledger | Heartbeats and per-event statistics from all agents |
| `actions` | Gateway, Ledger | Ingestor, Ledger | Human approval/denial decisions and ingestor timestamp coordination |

All topics are auto-created by agents via `ensure_topic()` if they do not already exist.

---

## Database Schema

The `logs` table is progressively enriched as each service processes an event. Related tables are keyed by `log_id` and inserted when the Ledger consumes from the corresponding topic.

Tables are created automatically by the Ledger on startup. You only need to create the `cybercontrol` database itself.

---

## Technology Stack

| Component | Technology |
|---|---|
| AI agents | Python 3.10+, OpenAI GPT-4.1 (`gpt-4.1`), LangChain |
| RAG embeddings | OpenAI `text-embedding-3-small` |
| Message broker | Apache Kafka with `kafka-python-ng` |
| Primary database | PostgreSQL |
| Log source | Elasticsearch, GNS2, Mock Generator |
| API layer | FastAPI, Pydantic, `psycopg2` |
| Frontend | React , TypeScript, Vite |
| Infrastructure | Docker, Docker Compose |

---

## Prerequisites

- Docker and Docker Compose
- An OpenAI API key
- An Elasticsearch instance reachable from the ingestor (or use the mock adapter for local testing)

---

## Getting Started

### 1. Create a workspace and clone all repositories

```bash
mkdir aurora && cd aurora

git clone https://github.com/Admin-or-Admin/.github.git
git clone https://github.com/Admin-or-Admin/ingestor.git
git clone https://github.com/Admin-or-Admin/rac-agents.git
git clone https://github.com/Admin-or-Admin/ledger.git
git clone https://github.com/Admin-or-Admin/gateway.git
git clone https://github.com/Admin-or-Admin/shared.git
git clone https://github.com/Admin-or-Admin/dashboard.git
git clone https://github.com/Admin-or-Admin/device_simulation.git
```

### 2. Copy the Docker Compose file

```bash
cp .github/docker-compose.yml .
```

### 3. Create a `.env` file

Create a `.env` file in the workspace root and add at minimum:

```env
OPENAI_API_KEY=sk-...
```

If you are deploying to a non-local environment, also configure any service URLs and broker addresses in `docker-compose.yml` and the Dashboard `.env` file.

### 4. Start the full stack

```bash
docker compose up --build -d
```

**Recommended startup order** (when starting services manually outside Docker):

```
1. docker compose up -f ./.github/docker-compose-externals-only.yml -d   # Kafka, PostgreSQL, Elasticsearch
2. ledger                        # ready before ingestor starts
3. ingestor                      # begins publishing to logs.unfiltered
4. classifier                   # begins consuming logs.unfiltered
5. analyst
6. responder
7. gateway
8. dashboard
```

### 5. Open the dashboard

When the services are manually started dashboard is available at:
```
http://localhost:5173
```
Or when using Docker Compose at:
```
http://localhost:8080
```
Gateway API documentation (Swagger UI) is available at:

```
http://localhost:8000/docs
```

---

## Configuration Reference

### Ingestor
| Variable | Code Default | Description |
|---|---|---|
| `ENABLED_INGESTORS` | `elasticsearch,mock_logs` | Comma-separated adapters to start. |
| `ELASTIC_HOST` | `http://localhost:9200` | Elasticsearch URL including scheme and port. |
| `ELASTIC_INDEX` | `mock-logs` | Index to read from/write to. |
| `KAFKA_BROKERS` | `localhost:29092` | Kafka broker address. |
| `KAFKA_TOPIC` | `logs.unfiltered` | Topic to publish events to. |
| `POLL_INTERVAL` | `1.0` | Seconds between Elasticsearch polls. |
| `MOCK_DELAY` | `0.5` | Base delay between mock log generation. |
| `SYSLOG_PORT` | `1514` | UDP port for GNS3 syslog listener. |
| `SIM_INTERVAL` | `1.0` | Seconds between simulated GNS3 network events. |
| `GNS3_SIMULATION_ENABLED` | `true` | Simulate GNS3 network events. |
| `CISCO_SYSLOG_PORT` | `1515` | Cisco UDP port for syslog listener. |
| `CISCO_SIMULATION_ENABLED` | `true` | Simulate Cisco network events. |
| `CISCO_SIM_INTERVAL` | `1.0` | Cisco simulation speed. |

### RAC Agents
| Variable | Code Default | Description |
|---|---|---|
| `OPENAI_API_KEY` | — | OpenAI key for GPT-4 and embeddings. |
| `KAFKA_BROKERS` | `localhost:29092` | Kafka broker address. |
| `CLASSIFIER_MODE` | `manual` | `manual` (stdin) or `kafka`. |
| `ANALYST_MODE` | `manual` | `manual` (reads pipeline) or `kafka`. |
| `RESPONDER_MODE` | `manual` | `manual` (reads pipeline) or `kafka`. |
| `MIN_CLASSIFICATION_CONFIDENCE` | `70` | Analyst drops logs below this confidence %. |
| `CLASSIFIER_HEARTBEAT_INTERVAL` | `30` | Seconds between classifier heartbeats. |
| `ANALYST_HEARTBEAT_INTERVAL` | `30` | Seconds between analyst heartbeats. |
| `RESPONDER_HEARTBEAT_INTERVAL` | `30` | Seconds between responder heartbeats. |
| `RATE_LIMIT_WAIT_SECONDS` | `60` | Wait time when LLM is rate limited. |
| `KNOWLEDGE_DIR` | `classifierKnowledge` | RAG folder for the classifier. |
| `KNOWLEDGE_CHUNK_SIZE` | `500` | Words per chunk (Classifier). |
| `KNOWLEDGE_CHUNK_OVERLAP` | `50` | Overlap (Classifier). |
| `KNOWLEDGE_TOP_K` | `5` | Chunks retrieved (Classifier). |
| `ANALYST_KNOWLEDGE_DIR` | `analystKnowledge` | RAG folder for the analyst. |
| `ANALYST_KNOWLEDGE_CHUNK_SIZE`| `500` | Words per chunk (Analyst). |
| `ANALYST_KNOWLEDGE_TOP_K` | `5` | Chunks retrieved (Analyst). |
| `RESPONDER_KNOWLEDGE_DIR` | `responderKnowledge` | RAG folder for the responder. |
| `RESPONDER_KNOWLEDGE_CHUNK_SIZE`| `500` | Words per chunk (Responder). |
| `RESPONDER_KNOWLEDGE_TOP_K` | `5` | Chunks retrieved (Responder). |

### Ledger
| Variable | Code Default | Description |
|---|---|---|
| `KAFKA_BROKERS` | `192.168.1.6:29092` | Kafka broker address. |
| `KAFKA_GROUP_ID` | `ledger-group` | Consumer group ID. |
| `DATABASE_URL` | `postgresql://admin:secret@localhost:5432/cybercontrol` | PostgreSQL connection string. |
| `LOG_LEVEL` | `INFO` | Logging verbosity. |
| `SCRAPE_FROM_BEGINNING` | `false` | If true, processes all Kafka history on start. |
| `DROP_DB` | `false` | If true, wipes the database on start. |
| `EXCLUDE_TOPICS` | `""` | Comma-separated list of topics to ignore. |

### Gateway
| Variable | Code Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql://admin:secret@localhost:5432/cybercontrol` | PostgreSQL connection string. |
| `PORT` | `8000` | API port. |
| `HOST` | `0.0.0.0` | API host. |
| `APP_TITLE` | `Aurora Gateway` | Swagger documentation title. |
| `APP_VERSION` | `1.0.0` | Swagger documentation version. |
| `OPENAI_API_KEY` | — | OpenAI key for the AI Analyst chat. |
| `OPENAI_MODEL` | `gpt-4.1` | Model used for chat/analysis. |
| `CORS_ORIGINS` | `http://localhost:5173,...` | Comma-separated list of allowed origins. |

### Dashboard
| Variable | Code Default | Description |
|---|---|---|
| `VITE_API_URL` | `http://localhost:8000` | URL of the running Gateway. |


---

# Aurora API Reference

### Logs
| Method | Endpoint | Description |
|---|---|---|
| GET | `/logs` | Paginated log list. Optional `?service_name=` filter |
| GET | `/logs/{log_id}` | Retrieve a specific raw log by its ID |
| GET | `/logs/{log_id}/details` | Log + classification + threat assessment |
| GET | `/logs/{log_id}/full` | Full record — log, classification, threat, remediation, and incident |

### Classifications
| Method | Endpoint | Description |
|---|---|---|
| GET | `/classifications` | Paginated list of all classifications |
| GET | `/classifications/{log_id}` | Classification for a specific log |

### Threats
| Method | Endpoint | Description |
|---|---|---|
| GET | `/threats` | Paginated list of all threat assessments |
| GET | `/threats/{log_id}` | Threat assessment for a specific log |

### Incidents
| Method | Endpoint | Description |
|---|---|---|
| GET | `/incidents` | Paginated list of all incidents (ordered by `resolved_at`) |
| GET | `/incidents/{incident_id}` | Single incident details |
| GET | `/incidents/{incident_id}/remediation` | All remediation steps for the incident's log |
| GET | `/incidents/{incident_id}/actions` | All follow-up actions for the incident |
| PATCH | `/incidents/{incident_id}` | Update status, executive summary, or outcome |
| POST | `/incidents/{incident_id}/comments` | Add a manual analyst comment to an incident |

### Remediation
| Method | Endpoint | Description |
|---|---|---|
| GET | `/remediation` | Paginated list of all remediation steps across all logs |
| GET | `/remediation/log/{log_id}` | All steps for a specific log, ordered by step number |
| PATCH | `/remediation/{step_id}` | Approve or deny a step. Body: `{"status": "approved"}` or `{"status": "denied"}` |

### Follow-up Actions
| Method | Endpoint | Description |
|---|---|---|
| GET | `/follow-ups` | Paginated list of all follow-up actions |
| GET | `/follow-ups/incident/{incident_id}` | All follow-up actions for a specific incident |

### Stats
| Method | Endpoint | Description |
|---|---|---|
| GET | `/stats/severity` | Classification counts grouped by severity |
| GET | `/stats/incidents/trend` | Incidents resolved per day (last 30 days) |
| GET | `/stats/services` | Log count per service name |
| GET | `/stats/top-attackers` | Entities with the most associated incidents |
| GET | `/stats/summary` | Aggregated dashboard summary (total logs, incidents, etc.) |

### AI Analyst (Chats)
| Method | Endpoint | Description |
|---|---|---|
| POST | `/chats/completion` | Simple LLM chat with optional log context |
| POST | `/chats/analyze` | Advanced Analyst Agent with tool-calling capabilities |
| GET | `/chats` | List all chat sessions |
| POST | `/chats` | Create a new chat session |
| GET | `/chats/{session_id}` | Get session details and full message history |
| POST | `/chats/{session_id}/messages` | Add a user/assistant message to a session |
| DELETE| `/chats/{session_id}` | Delete a chat session |

### System Health
| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Status of all Aurora microservices and database connection |
| GET | `/metrics` | High-level system performance and volume metrics |

**Pagination:** All list endpoints (e.g., `/logs`, `/incidents`) accept `?limit=100&offset=0` query parameters (max limit 1000).


---

## Running Services Individually

### Ingestor (Python)

```bash
cd ingestor
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in your values

# Run from the parent directory so shared/ imports resolve
PYTHONPATH=. python -m ingestor.main

# Run with a specific adapter set
ENABLED_INGESTORS=mock_logs PYTHONPATH=. python -m ingestor.main
ENABLED_INGESTORS=gns3 PYTHONPATH=. python -m ingestor.main
```

### RAC Agents (Python)

```bash
cd rac-agents
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env .env.backup && cp .env.example .env   # fill in keys and Kafka address

mkdir -p classifierKnowledge analystKnowledge responderKnowledge

# Start each agent in a separate terminal (all require venv activated)
python classifier.py
python analyst.py
python responder.py
```

### Ledger (Python)

```bash
cd ledger
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in DATABASE_URL and KAFKA_BROKERS

# Create the database (skip if using Docker Compose — it's created automatically)
psql -U admin -c "CREATE DATABASE cybercontrol;"

python main.py
```

### Gateway (Python)

```bash
cd gateway
python -m venv venv && source venv/bin/activate

# Install shared library first
pip install ./shared/
pip install -r requirements.txt

cp .env.example .env   # fill in DATABASE_URL and OPENAI_API_KEY
python main.py
```

### Dashboard (Node.js)

```bash
cd dashboard
npm install
cp .env.example .env   # set VITE_API_URL=http://localhost:8000
npm run dev
```

---

## Manual Mode (No Kafka)

Set all three agent mode variables to `manual` in `.env` and run them sequentially in a single terminal to test a specific log end-to-end without Kafka.

```env
CLASSIFIER_MODE=manual
ANALYST_MODE=manual
RESPONDER_MODE=manual
```

```bash
# Step 1 — classify a log interactively
python classifier.py
LOG > 2026-01-15 14:23:01 Failed login attempt for user admin from 192.168.1.105
# Classification is printed and written to pipeline.json

# Step 2 — investigate
python analyst.py
# Reads pipeline.json, investigates if it passes filters, writes investigation back

# Step 3 — generate remediation plan
python responder.py
# Reads pipeline.json, prints the full resolution plan
```

`pipeline.json` is the shared state file for manual mode. It accumulates all pipeline stages in a single file and is listed in `.gitignore`. Never commit it.

---

## Extending Aurora

### Adding a New Log Source

Create a new adapter in `ingestor/adapters/`:

```python
from .base import BaseAdapter
from shared.kafka_client import AuroraProducer
import os, time

class SplunkAdapter(BaseAdapter):
    def __init__(self, name="Splunk"):
        super().__init__(name)
        self.kafka_brokers = [os.getenv("KAFKA_BROKERS", "localhost:29092")]
        self.kafka_topic   = os.getenv("KAFKA_TOPIC", "logs.unfiltered")

    def run(self):
        producer = AuroraProducer(self.kafka_brokers)
        producer.ensure_topic(self.kafka_topic)
        while True:
            events = fetch_from_splunk()
            for event in events:
                producer.send_log(self.kafka_topic, event)
            producer.flush()
            time.sleep(5)
```

Register it in `ingestor/main.py` under `ADAPTER_MAP`, then enable it with `ENABLED_INGESTORS=splunk` in `.env`. Every published document must include at minimum `@timestamp`, `message`, and `service.name`.

### Adding a New Gateway Endpoint

1. Add a Pydantic schema in `gateway/schemas.py`
2. Create a route file in `gateway/routes/`
3. Register the router in `gateway/main.py` with `app.include_router()`

### Adding Agent Knowledge

Copy files into the relevant knowledge folder and restart the agent. Supported formats: `.pdf`, `.docx`, `.txt`, `.md`. The agent will re-embed only changed files on the next startup.

---

## Project Structure

```
aurora/                            — workspace root
  docker-compose.yml               — full stack orchestration

  ingestor/
    main.py                        — adapter registry and startup coordination
    adapters/
      base.py                      — BaseAdapter interface
      elasticsearch_adapter.py     — production log source with cursor-based polling
      mock_logs_adapter.py         — synthetic log generator for testing
      gns3_adapter.py              — UDP syslog listener for GNS3/Network events
      cisco_adapter.py             — Cisco-specific syslog listener and simulator
    Dockerfile                     — container definition
    requirements.txt               — python dependencies

  rac-agents/
    classifier.py                  — Agent 1: classifies raw logs
    analyst.py                     — Agent 2: investigates security events
    responder.py                   — Agent 3: generates remediation plans
    llm_router.py                  — LLM strategy and key rotation
    kafka_client.py                — specialized producer/consumer wrappers
    classifierKnowledge_loader.py  — RAG loader for the classifier
    analystKnowledge_loader.py     — RAG loader for the analyst
    responderKnowledge_loader.py   — RAG loader for the responder
    classifierKnowledge/           — directory for classifier RAG data
    analystKnowledge/              — directory for analyst RAG data
    responderKnowledge/            — directory for responder RAG data
    pipeline.json                  — local shared state (manual mode)
    Dockerfile                     — container definition
    requirements.txt               — python dependencies

  ledger/
    main.py                        — unified Kafka consumer loop
    config.py                      — configuration and environment management
    database.py                    — PostgreSQL schema and DB operations
    handlers.py                    — logic for processing specific topic messages
    Dockerfile                     — container definition
    requirements.txt               — python dependencies

  gateway/
    main.py                        — FastAPI application and router setup
    config.py                      — application configuration
    database.py                    — database connection and query execution
    schemas.py                     — Pydantic models (DTOs)
    core/
      crud.py                      — base CRUD operations
    routes/
      logs.py                      — log retrieval endpoints
      classifications.py           — classification endpoints
      threats.py                   — threat assessment endpoints
      incidents.py                 — incident management endpoints
      remediation.py               — remediation step endpoints
      follow_ups.py                — follow-up action endpoints
      stats.py                     — metrics and dashboard stats endpoints
      chats.py                     — AI Analyst chat and tool-calling endpoints
      health.py                    — system health and metrics endpoints
    Dockerfile                     — container definition
    requirements.txt               — python dependencies

  shared/                          — local python library used by all services
    kafka_client.py                — shared Kafka utilities
    elastic_client.py              — shared Elasticsearch client
    logger.py                      — standardized logging configuration
    setup.py                       — installation script for internal use

  dashboard/
    src/
      App.tsx                      — main layout and routing
      main.tsx                     — application entry point
      index.css                    — global styles and design system
      lib/
        api.ts                     — API client and TypeScript interfaces
      components/
        ui.tsx                     — reusable UI components
        ChatSidebar.tsx            — sidebar for AI Analyst chat history
      pages/
        Overview.tsx               — dashboard landing page
        LogStream.tsx              — real-time log viewer
        Threats.tsx                — threat assessment list
        Incidents.tsx              — incident management
        Remediation.tsx            — remediation workflow
        AgentMonitor.tsx           — real-time agent status
        AIAnalyst.tsx              — interactive AI agent interface
        AgentChat.tsx              — log-specific chat interface
    Dockerfile                     — container definition
    package.json                   — frontend dependencies and scripts
    vite.config.ts                 — build configuration

  device_simulation/               — extra utility for network device testing
    main.py                        — simulation logic
    Dockerfile                     — container definition

```

---

## Related Repositories

| Repository | Description |
|---|---|
| [`ingestor`](https://github.com/Admin-or-Admin/ingestor) | Log ingestion adapters (Elasticsearch, Mock, GNS3, Cisco) |
| [`rac-agents`](https://github.com/Admin-or-Admin/rac-agents) | Classifier, Analyst, Responder agents |
| [`ledger`](https://github.com/Admin-or-Admin/ledger) | Kafka consumer that writes all pipeline events to PostgreSQL |
| [`gateway`](https://github.com/Admin-or-Admin/gateway) | FastAPI REST API |
| [`shared`](https://github.com/Admin-or-Admin/shared) | Shared client libraries used by Python services |
| [`dashboard`](https://github.com/Admin-or-Admin/dashboard) | React + TypeScript operator dashboard |
| [`device_simulation`](https://github.com/Admin-or-Admin/device_simulation) | Network device simulation for testing |
| [`.github`](https://github.com/Admin-or-Admin/.github) | Docker Compose files |

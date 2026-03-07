# CyberControl

**AI-Powered Cybersecurity Control Center**

CyberControl is an open, event-driven platform that ingests logs from any observability source, classifies them using AI, and autonomously investigates and responds to cybersecurity threats — with a human always in control of the final decision.

Built on Apache Kafka as its central nervous system, the platform is composed of seven independent microservices that can be deployed, scaled, and replaced individually. Each service communicates exclusively through Kafka topics, making the architecture fully decoupled and production-ready from day one.

---

## Repositories

| Repository | Role | Language |
|------------|------|----------|
| [`ingestor`](#ingestor) | Pulls logs from any source into the pipeline | Node.js |
| [`classifier`](#classifier) | Categorises and tags every log using AI | Node.js |
| [`analyst`](#analyst) | Investigates security events with full cross-domain context | Node.js |
| [`responder`](#responder) | Executes remediation steps, manages human approvals | Node.js |
| [`ledger`](#ledger) | Persists all pipeline events to the database | Node.js |
| [`gateway`](#gateway) | REST + WebSocket API serving the frontend | Node.js |
| [`console`](#console) | Operator dashboard for monitoring and incident response | TypeScript / React |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ANY INPUT SYSTEM                             │
│              Elasticsearch · Splunk · Syslog · Files                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                        [ ingestor ]
                             │
                             ▼
                    ┌─────────────────┐
                    │  logs.unfiltered │
                    └────────┬────────┘
                             │
                       [ classifier ]
                             │
                             ▼
                    ┌─────────────────┐
                    │ logs.categories  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
        [ analyst ]                    [ ledger ]
              │                             │
              ▼                             ▼
    ┌──────────────────┐               ┌────────┐
    │ logs.solver_plan  │               │   DB   │
    └────────┬─────────┘               └────┬───┘
             │                              │
      [ responder ]                   [ gateway ]
             │                              │
             ▼                             ▼
      ┌──────────────┐              ┌─────────────┐
      │ logs.solution │              │   console   │
      └──────┬───────┘              └─────────────┘
             │
        [ ledger ]
```

---

## Kafka Topics

All inter-service communication happens exclusively through the following Kafka topics. No service calls another service directly.

| Topic | Producer | Consumers | Description |
|-------|----------|-----------|-------------|
| `logs.unfiltered` | `ingestor` | `classifier`, `ledger` | Raw log documents exactly as received from the source system |
| `logs.categories` | `classifier` | `analyst`, `ledger` | Logs enriched with `category`, `severity`, `tags`, and `isCybersecurity` |
| `logs.solver_plan` | `analyst` | `responder`, `ledger` | Security logs enriched with AI threat analysis and a proposed step-by-step remediation plan |
| `logs.solution` | `responder` | `ledger` | Completed incident records including step execution results and effectiveness verification |
| `actions` | `responder`, `gateway` | `responder` | Human approval and denial decisions, plus agent auto-execution records |

---

## Data Flow

A single security event travels through the full pipeline in under 10 seconds from the moment it appears in the source system to the moment a remediation plan is ready for human review.

```
14:32:11.000  Event appears in source system
14:32:11.423  → logs.unfiltered    Raw document ingested and published
14:32:13.891  → logs.categories   Classified as security / critical (2.4s)
14:32:19.234  → logs.solver_plan  Investigated, plan with 4 steps ready (5.3s)
14:34:01.882  → actions           Human approves steps via console
14:35:44.120  → logs.solution     Incident resolved, effectiveness verified

Total: 3 minutes 33 seconds from detection to resolution
```

---

## Services

### `ingestor`

Connects to any log source and streams raw events into the Kafka pipeline. Designed to be source-agnostic — the adapter pattern means adding a new source (Splunk, CloudWatch, raw syslog) requires only a new adapter file without touching the core pipeline logic.

**Current adapters:** Elasticsearch

**Key responsibilities:**
- Maintains a cursor (last seen timestamp) to avoid re-processing logs on restart
- Publishes to `logs.unfiltered` with full source metadata preserved
- Handles backpressure gracefully when Kafka is slow

**Environment variables:**
```
ELASTIC_HOST=http://your-elastic-host:9200
ELASTIC_API_KEY=your_api_key
ELASTIC_INDEX_PATTERN=logs-*
POLL_INTERVAL_MS=15000
KAFKA_BROKERS=localhost:9092
```

---

### `classifier`

Consumes raw logs from `logs.unfiltered` and uses an AI model to assign each log a `category`, `severity`, `tags`, and a `isCybersecurity` boolean. Processes logs in configurable batches for efficiency.

**Categories assigned:** `security` · `infrastructure` · `application` · `deployment`

**Severity levels:** `critical` · `high` · `medium` · `low` · `info`

**Key responsibilities:**
- Batches logs before calling the AI model to reduce API costs
- Falls back to rule-based classification if the AI call fails
- Records `classificationConfidence` so low-confidence results can be flagged for human review
- Records `reasoning` so every classification decision is fully explainable

**Environment variables:**
```
KAFKA_BROKERS=localhost:9092
KAFKA_GROUP_ID=classifier-group
ANTHROPIC_API_KEY=your_key
BATCH_SIZE=20
```

---

### `analyst`

The core intelligence service. Consumes only security-flagged logs from `logs.categories` and performs deep threat analysis using a 1-hour rolling window of all logs — including infrastructure, application, and deployment logs — as cross-domain context.

This cross-domain context is the key differentiator of the platform. The analyst can identify that a brute force attack began 14 minutes after a new deployment, or that a port scan correlates with an unusual spike in database connection failures — correlations that a system looking only at security logs would miss entirely.

**Key responsibilities:**
- Maintains an in-memory rolling context window of all recent logs (not just security)
- Correlates the security event against the full context to find related events
- Produces a structured remediation plan with per-step risk levels and rollback instructions
- Flags individual steps as `autoExecute: true` (safe to run without human approval) or `requiresApproval: true`
- Records `correlationInsight` explaining why specific logs were linked to the event

**Environment variables:**
```
KAFKA_BROKERS=localhost:9092
KAFKA_GROUP_ID=analyst-group
ANTHROPIC_API_KEY=your_key
CONTEXT_WINDOW_MINUTES=60
```

---

### `responder`

Consumes remediation plans from `logs.solver_plan` and manages the full resolution lifecycle. For steps marked `autoExecute: true`, it executes immediately. For steps marked `requiresApproval: true`, it waits for a human decision to arrive on the `actions` topic before proceeding.

After all steps are executed, it queries the source system to verify the fix actually worked and records the result in `logs.solution`.

**Key responsibilities:**
- Maintains a command whitelist — only pre-approved command prefixes can be executed, never arbitrary strings
- Supports partial plan execution — a human can approve some steps and deny others
- Verifies fix effectiveness by re-querying the source system after execution
- Publishes a full resolution record to `logs.solution` including outcome, duration, and affected accounts

**Whitelisted command prefixes:**
```
kubectl rollout restart
kubectl scale deployment
kubectl set env
aws wafv2 update-ip-set
aws ec2 revoke-security-group-ingress
certbot renew
```

**Environment variables:**
```
KAFKA_BROKERS=localhost:9092
KAFKA_GROUP_ID=responder-group
ELASTIC_HOST=http://your-elastic-host:9200
ELASTIC_API_KEY=your_key
```

---

### `ledger`

The only service that writes to the database. Subscribes to all five Kafka topics and upserts each message into the central `logs` table as it progresses through the pipeline. A single log row is progressively enriched — starting as a raw document and ending as a fully resolved incident record.

**Key responsibilities:**
- Subscribes to all topics: `logs.unfiltered`, `logs.categories`, `logs.solver_plan`, `logs.solution`, `actions`
- Upserts on `log.id` — the same database row is enriched as the log moves through each pipeline stage
- Tracks `processingStage` so the current state of any log is always queryable
- Never exposes the database externally — all reads go through `gateway`

**Environment variables:**
```
KAFKA_BROKERS=localhost:9092
KAFKA_GROUP_ID=ledger-group
DATABASE_URL=postgresql://admin:secret@postgres:5432/cybercontrol
```

---

### `gateway`

The single entry point between the outside world and the platform internals. Exposes a REST API for querying logs, services, and analytics, and a WebSocket connection for real-time log push to connected clients. Receives human approval and denial decisions from the `console` and publishes them to the `actions` Kafka topic.

**Key responsibilities:**
- Serves security logs to the UI (filtered from the full log set)
- Pushes newly investigated security events to all connected WebSocket clients in real time
- Receives human decisions (`approve` / `deny`) and forwards them to the `actions` topic
- Provides health and analytics endpoints for the dashboard

**REST endpoints:**
```
GET  /api/logs              Security logs for the UI (categorised + investigated)
GET  /api/logs/all          All logs (used for agent context display)
GET  /api/logs/:id          Single log with full investigation data
GET  /api/services/health   Service health metrics
GET  /api/analytics         Aggregated statistics for the analytics page
POST /api/agent/approve     Submit a human approval decision for a step
POST /api/agent/deny        Submit a human denial decision for a step

WS   /ws                    Real-time log push to connected console clients
```

**Environment variables:**
```
KAFKA_BROKERS=localhost:9092
DATABASE_URL=postgresql://admin:secret@postgres:5432/cybercontrol
PORT=3001
CORS_ORIGIN=http://localhost:5173
```

---

### `console`

The operator-facing dashboard. Displays real-time security events, AI investigation results, and guided remediation workflows. Built as a single-page application that connects to `gateway` via REST on load and WebSocket for live updates.

**Pages:**
- **Overview** — KPI cards, error rate charts, severity distribution, AI agent activity feed
- **Log Stream** — Live security event table with expandable AI analysis per event
- **AI Agent** — Step-by-step remediation plan with approve/deny controls and agent chat console
- **Analytics** — Hourly breakdown, weekly trends, AI effectiveness metrics, service rankings
- **Services** — Real-time health cards for all monitored services
- **Pipeline** — Kafka topic flow diagram with consumer lag and throughput metrics
- **Alerts** — Active incidents with acknowledgement controls
- **Rules** — Detection rule management
- **Settings** — Source system configuration, AI model selection, notification routing

**Environment variables:**
```
VITE_API_URL=http://localhost:3001
VITE_WS_URL=ws://localhost:3001
```

---

## Running the Full Platform

### Prerequisites

- Docker and Docker Compose
- An Elasticsearch instance with logs
- An Anthropic API key

### Quick Start

```bash
# 1. Clone the organisation
git clone https://github.com/your-org/ingestor
git clone https://github.com/your-org/classifier
git clone https://github.com/your-org/analyst
git clone https://github.com/your-org/responder
git clone https://github.com/your-org/ledger
git clone https://github.com/your-org/gateway
git clone https://github.com/your-org/console

# 2. Copy and fill in environment variables
cp .env.example .env
# Edit .env with your Elasticsearch host, API keys, etc.

# 3. Start everything
docker-compose up --build

# 4. Open the console
open http://localhost:5173
```

### Starting Services Individually

Each service is independently runnable:

```bash
cd ingestor && npm install && npm run dev
cd classifier && npm install && npm run dev
cd analyst && npm install && npm run dev
cd responder && npm install && npm run dev
cd ledger && npm install && npm run dev
cd gateway && npm install && npm run dev
cd console && npm install && npm run dev
```

### Recommended Build Order

When setting up for the first time, start services in this order to validate each layer before adding the next:

```
1. ingestor    → verify logs appear in logs.unfiltered
2. classifier  → verify logs appear in logs.categories with correct classification
3. ledger      → verify logs are being persisted to the database
4. gateway     → verify the API returns real data
5. console     → verify the UI displays real logs
6. analyst     → verify AI investigation plans appear on log expansion
7. responder   → verify step execution and resolution records
```

---

## Database Schema

The central `logs` table is progressively enriched as each service processes a log entry.

```sql
CREATE TABLE logs (
    id                  VARCHAR PRIMARY KEY,
    elastic_index       VARCHAR,
    timestamp           TIMESTAMPTZ,
    message             TEXT,
    service             VARCHAR,
    host                VARCHAR,
    environment         VARCHAR,
    raw_severity        VARCHAR,

    -- Added by classifier
    category            VARCHAR,
    severity            VARCHAR,
    tags                JSONB,
    is_cybersecurity    BOOLEAN,
    classification_confidence INTEGER,
    classification_reasoning  TEXT,

    -- Added by analyst
    ai_suggestion       TEXT,
    attack_vector       TEXT,
    recurrence_rate     INTEGER,
    complexity          VARCHAR,
    auto_fixable        BOOLEAN,
    requires_human_approval BOOLEAN,
    proposed_steps      JSONB,
    correlated_log_ids  JSONB,
    correlation_insight TEXT,
    notify_teams        JSONB,

    -- Added by responder
    resolution_outcome  VARCHAR,
    resolved_by         VARCHAR,
    human_approved_by   VARCHAR,
    step_results        JSONB,
    incident_id         VARCHAR,

    -- Pipeline tracking
    processing_stage    VARCHAR,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Message Schema Versioning

Every Kafka message includes a `meta.schemaVersion` field. When a breaking schema change is needed, bump the version and run both consumers in parallel during the transition window before decommissioning the old version. This ensures zero-downtime schema migrations.

---

## Adding a New Log Source

To add a new source to `ingestor`, create a new adapter file:

```
ingestor/
└── src/
    └── adapters/
        ├── elasticsearch.ts   ← existing
        ├── splunk.ts          ← add this
        └── cloudwatch.ts      ← add this
```

Each adapter must implement the `LogAdapter` interface:

```typescript
interface LogAdapter {
  name: string;
  fetchNewLogs(): Promise<RawLog[]>;
}
```

The core polling loop in `ingestor` calls `fetchNewLogs()` on every configured adapter on each tick, normalises the results, and publishes them to `logs.unfiltered`. No other service needs to change.

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Message broker | Apache Kafka |
| Database | PostgreSQL |
| All services | Node.js + TypeScript |
| AI models | Anthropic Claude (Sonnet for analysis, Haiku for classification) |
| Frontend framework | React + TypeScript + Vite |
| Container orchestration | Docker Compose |
| Elasticsearch client | `@elastic/elasticsearch` |
| Kafka client | `kafkajs` |
| HTTP framework | Express |
| Real-time push | WebSockets (`ws`) |

---

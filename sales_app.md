# 🚀 PocketSales — Technical Pitch
## Senior Data Engineer | Fintech | Cloud-Native Data Platform

> **Javier López Ramírez** · Backend & Data Engineer · GCP Specialist  
> *Building production-grade data infrastructure from scratch*

---

## 📌 The Problem I Solved

Food & beverage businesses needed a **real-time sales + inventory platform** capable of:
- Tracking stock levels across multiple stores with **zero negative inventory**
- Answering *"what can I sell right now?"* in **< 100ms**
- Projecting all operational data into a **analytics warehouse** (BigQuery) with full audit trail
- Running on a tight budget → **serverless, pay-per-use architecture** on GCP

---

## 🏗️ High-Level System Architecture

```mermaid
graph TD
    subgraph "Client Layer"
        APP["📱 React Native + Expo\nCross-platform Mobile/Web"]
    end

    subgraph "API Layer — Cloud Run (FastAPI / Python 3.12)"
        CMD["📝 Command Handler\nWrite Operations"]
        QRY["📖 Query Handler\nRead Operations"]
        AUTH["🔐 Firebase Auth + RBAC\nJWT Validation per request"]
    end

    subgraph "Operational Store — Firestore (NoSQL)"
        WM["Write Models\nNormalized · Transactional\nbusiness_id scoped"]
        RM["Read Models (rm_*)\nDenormalized · Pre-computed\n~30–50ms query latency"]
        OUTBOX["Audit Outbox\nevents_processed / commands_processed"]
    end

    subgraph "Event Bus — Google Pub/Sub"
        PS["Topic: events\nAt-least-once delivery\nIdempotency enforced"]
    end

    subgraph "ETL / Projection Layer — Cloud Functions Gen2"
        RMP["🔄 Read Model Projector\nPython 3.12 · 512MB RAM"]
        ICH["📦 Inventory Commands Handler"]
        TGS["💾 Transactions → GCS Exporter"]
    end

    subgraph "Data Warehouse — Google BigQuery"
        BQ_TH["transaction_header\nPartitioned by event_date\nClustered by store_id"]
        BQ_TL["transaction_line\nDetailed line items"]
        BQ_MI["menu_item\nCatalog snapshot"]
        BQ_IV["inventory_valuation\nDaily stock snapshots"]
    end

    subgraph "Infrastructure — IaC"
        TF["Terraform\nGCP Resources"]
        CB["Cloud Build\nCI/CD Pipelines"]
    end

    APP -->|"REST API (HTTPS)"| CMD
    APP -->|"REST API (HTTPS)"| QRY
    AUTH -.->|"validates"| CMD
    AUTH -.->|"validates"| QRY

    CMD --> WM
    CMD --> OUTBOX
    CMD -->|"Atomically"| PS
    QRY --> RM

    PS --> RMP
    PS --> ICH
    PS --> TGS

    RMP -->|"Project"| RM
    ICH -->|"Mutate"| WM
    TGS -->|"Load"| BQ_TH
    TGS -->|"Load"| BQ_TL

    WM -.->|"snapshot"| BQ_MI
    BQ_TH --> BQ_IV

    TF -.->|"provisions"| PS
    TF -.->|"provisions"| BQ_TH
    CB -.->|"deploys"| RMP
```

---

## 📊 CQRS + Read Model Pattern: The Data Engineering Core

The most critical design decision was separating **Write Models** (normalized, ACID-safe) from **Read Models** (denormalized, query-optimized).

### Before vs. After

```mermaid
graph LR
    subgraph "❌ Before — 4 Queries + In-Memory JOIN"
        Q1["Query menu_items\n~50ms"] --> Q2["Query stock_items\n~50ms"]
        Q2 --> Q3["Query inventory_stocks\n~50ms"]
        Q3 --> Q4["Query components\n~100ms"]
        Q4 --> JOIN["JOIN 202 components\n~500ms"]
        JOIN --> CALC["Calculate availability\n~1000ms"]
        CALC --> R1["⏱️ Total: 2–5s ❌"]
    end

    subgraph "✅ After — Single Pre-Computed Query"
        QRM["Query rm_pos_availability\n1 collection"] --> R2["⏱️ Total: ~50ms ✅"]
    end
```

> **Result: 40–100x latency reduction** on the most critical endpoint (`GET /pos/availability`)

---

## 🔄 ETL Pipeline: From Events to Data Warehouse

This is the core **data engineering pipeline** I designed and implemented end-to-end.

```mermaid
sequenceDiagram
    participant API as FastAPI (Cloud Run)
    participant FS as Firestore (Write Model)
    participant PS as Pub/Sub (Event Bus)
    participant RMP as Read Model Projector (CF)
    participant GCS as Cloud Storage
    participant BQ as BigQuery

    rect rgb(220, 240, 255)
        Note over API,FS: WRITE — Atomic Transaction (Outbox Pattern)
        API->>FS: Write business document
        API->>FS: Write audit event (events_processed)
        FS-->>API: Commit (atomic)
        API->>PS: Publish event envelope {event_id, type, payload}
        API-->>API: 201 OK (~100ms)
    end

    rect rgb(255, 245, 220)
        Note over PS,RM: PROJECTION — Async Event Processing
        PS->>RMP: Trigger (Pub/Sub push)
        RMP->>FS: Check rm_events_processed (idempotency)
        alt New event
            RMP->>FS: Read source documents
            RMP->>FS: Write rm_* collection (denormalized)
            RMP->>FS: Mark event processed
        else Already processed
            RMP->>RMP: Skip safely (idempotent)
        end
    end

    rect rgb(220, 255, 230)
        Note over PS,BQ: WAREHOUSE LOAD — Analytics ETL
        PS->>GCS: transactions_to_gcs exports JSON
        GCS->>BQ: BigQuery Load Job
        Note over BQ: Partitioned by event_date\nClustered by store_id
    end
```

---

## 🗄️ Data Model: Multi-Tenant Hierarchy

```mermaid
erDiagram
    BUSINESS {
        string business_id PK
        string name
        string business_type "RETAIL / BOM / MIXED"
        bool is_active
    }
    STORE {
        string store_id PK
        string business_id FK
        string name
        bool is_active
    }
    TENANT_USER {
        string uid PK
        string business_id FK
        string[] roles "ADMIN / MANAGER / CASHIER / WAITER"
        string[] store_ids
    }
    STOCK_ITEM {
        string stock_item_id PK "STK-{hash}"
        string name
        string unit
    }
    INVENTORY_STOCK {
        string id PK "biz__store__stk"
        string business_id FK
        string store_id FK
        float qty_on_hand
        float reserved
    }
    INVENTORY_MOVEMENT {
        string movement_id PK "MOV-{hash}"
        string type "IN / OUT / ADJUST"
        float qty_delta
        timestamp created_at
    }
    SALE_HEADER {
        string sale_id PK
        float total_amount
        string payment_method
        timestamp event_date
        string store_id FK
    }
    SALE_LINE {
        string line_id PK
        string sale_id FK
        string stock_item_id FK
        float qty
        float unit_price
    }

    BUSINESS ||--o{ STORE : "has"
    BUSINESS ||--o{ TENANT_USER : "belongs to"
    STORE ||--o{ INVENTORY_STOCK : "tracks"
    STOCK_ITEM ||--o{ INVENTORY_STOCK : "stocked in"
    INVENTORY_STOCK ||--o{ INVENTORY_MOVEMENT : "records"
    STORE ||--o{ SALE_HEADER : "generates"
    SALE_HEADER ||--o{ SALE_LINE : "contains"
    STOCK_ITEM ||--o{ SALE_LINE : "sold as"
```

> **Key constraint**: All queries are **scoped by `business_id` + `store_id`**. Tenant data isolation is enforced at the service layer and at Firestore rules level. Cross-tenant data leakage is classified as a **Sev-0 incident**.

---

## ⚙️ Read Models Inventory (Active Projections)

| Read Model | Docs | Latency | Source Events |
|---|---|---|---|
| `rm_pos_availability` | 54 | ~50ms | `inventory_movement.*` |
| `rm_inventory_items` | 41 | ~30ms | `stock_item.*`, `inventory_movement.*` |
| `rm_menu_item_bom` | 202 | ~40ms | `menu_item_components.*` |
| `rm_bom_index` | 202 | ~40ms | `menu_item_components.*` |
| `rm_tx_headers` | N | ~30ms | `inventory.checkout_completed` |
| `rm_tx_lines` | N | ~30ms | `inventory.checkout_completed` |

---

## 🚰 CI/CD Pipeline: 4-Gate Quality Gate

Designed a **4-gate progressive validation pipeline** to ensure code quality, test coverage, and production readiness before every deployment.

```mermaid
graph LR
    PUSH["🔀 Push / PR"] --> A

    subgraph "Gate A — Fast Feedback  < 2 min"
        A["🔍 Lint + Unit Tests\nruff · pytest -m unit\nEvery push"]
    end

    subgraph "Gate B — Local E2E  5–10 min"
        B["🧪 Emulator Integration\nFirebase Emulators\npytest -m e2e\nPR → development"]
    end

    subgraph "Gate C — Dev Integration  10–15 min"
        C["☁️ Live GCP Dev\nCloud Build Wait\nPlaywright E2E\nPush → development"]
    end

    subgraph "Gate D — Production Readiness  15–20 min"
        D["🔐 Security + Release\npip-audit · npm audit\nVersion bump check\nCHANGELOG validation\nPR → main"]
    end

    A -->|"✅ Pass"| B
    B -->|"✅ Pass"| C
    C -->|"✅ Pass"| D
    D -->|"✅ Release"| DEPLOY["🚀 Cloud Run Deployment\n+ Cloud Functions Deploy"]
```

**Security**: CI/CD uses **Workload Identity Federation (WIF)** — zero long-lived secrets. GitHub Actions authenticates to GCP via OIDC token.

---

## 🔐 RBAC Security Model

```mermaid
graph TD
    subgraph "Role Hierarchy"
        ADMIN["👑 ADMIN\nFull access wildcard *"]
        MANAGER["🧑‍💼 MANAGER\nPOS + Full Inventory"]
        CASHIER["💳 CASHIER\nPOS + Transactions Read"]
        WAITER["🍽️ WAITER\nPOS Only"]
    end

    subgraph "Permission Matrix"
        P1["pos.sell"]
        P2["transactions.read"]
        P3["inventory.read"]
        P4["inventory.write"]
        P5["admin.settings"]
    end

    ADMIN --> P1 & P2 & P3 & P4 & P5
    MANAGER --> P1 & P2 & P3 & P4
    CASHIER --> P1 & P2
    WAITER --> P1
```

Enforcement is **dual-layer**:
- **Backend**: `require_roles(user, ["MANAGER", "ADMIN"])` on every FastAPI endpoint
- **Frontend**: `can("inventory.write")` gates UI components in React Native

---

## 🤖 AI Engineering Layer — PocketSales Agent

On top of the data platform, I built **pocketsales-agent**: an autonomous LLM-powered engineering assistant that operates on the codebase itself.

```mermaid
graph LR
    subgraph "Input"
        ISSUE["📋 GitHub Issue\nor natural language task"]
    end

    subgraph "pocketsales-agent"
        PARSER["🧩 Issue Parser\nExtracts structure & target file"]
        PLANNER["🗺️ Planner\nGenerates fix strategy"]
        EXEC["⚙️ Executor\nApplies code changes"]
        EVAL["✅ Evaluator\nBoolean validation (strict)"]
        BENCH["📊 Stability Benchmark\nFisher Score · Risk Model"]
    end

    subgraph "Observability"
        DATASET["Risk Dataset\nEarly signals (Un, Vn_hat)\nvs full outputs (Sf_diff)"]
    end

    ISSUE --> PARSER --> PLANNER --> EXEC --> EVAL
    EVAL -->|"loop until pass"| EXEC
    EVAL -->|"done"| BENCH
    BENCH --> DATASET
```

- Uses **Fisher Information** to measure solution stability across temperature regimes
- Builds a **contrastive risk dataset** (Phase 10) to train a predictive risk model
- Strict **contract alignment** validation prevents prompt drift (ADR-001)

---

## 🎯 Mapping to the Job Requirements

| Requirement | What I Built |
|---|---|
| **Scalable data pipelines & ETL** | 3 Cloud Functions: Read Model Projector, Inventory Commands Handler, Transactions → GCS → BigQuery ETL |
| **Data models for analytics** | BigQuery: `transaction_header` (partitioned + clustered), `transaction_line`, `menu_item`, `inventory_valuation` |
| **Data quality & idempotency** | `rm_events_processed` collection — every event idempotent via `event_id`; Outbox pattern for dual-write safety |
| **Data governance** | Full audit trail via Outbox; RBAC enforced at API + Firestore rules; soft-delete only |
| **Python / SQL** | FastAPI backend in Python 3.12; BigQuery SQL (partitioned, clustered); Pydantic schemas |
| **Cloud (GCP)** | Cloud Run · Firestore · Pub/Sub · Cloud Functions Gen2 · BigQuery · Cloud Build · Secret Manager |
| **CI/CD** | 4-gate GitHub Actions pipeline with Workload Identity Federation |
| **Data warehousing & ETL best practices** | CQRS pattern, event-driven projections, pre-computed read models, 40–100x query improvement |
| **IaC** | Terraform for GCP resource provisioning; Cloud Build YAML pipelines per environment |
| **Git discipline** | Feature branches → `development` → `main`; gated PRs; conventional commits; version bumps enforced |

---

## 📈 Measured Outcomes

| Metric | Before | After |
|---|---|---|
| POS availability query latency | 2–5 seconds | **~50ms** (40–100x improvement) |
| Query count per `/pos/availability` | 4 queries + in-memory JOIN | **1 query** |
| CPU load on API tier | High (compute-intensive JOIN) | **Near zero** (read-only) |
| Scale behavior | Degrades as data grows | **Constant latency** (async projection) |
| Audit coverage | None | **100%** of mutations via Outbox |
| Test gates before production | None | **4 progressive gates** |

---

## 🛠️ Full Technology Stack

```mermaid
mindmap
  root((PocketSales\nData Stack))
    Compute
      Cloud Run
        FastAPI Python 3.12
        Uvicorn ASGI
      Cloud Functions Gen2
        Read Model Projector
        Inventory Handler
        Transactions Exporter
    Storage
      Firestore
        Write Models normalized
        Read Models denormalized
        Audit Outbox
      Google BigQuery
        Partitioned tables
        Clustered by store_id
      Cloud Storage GCS
        ETL staging layer
    Messaging
      Pub/Sub
        At-least-once delivery
        Idempotent consumers
    Infrastructure
      Terraform IaC
      Cloud Build CI/CD
      Secret Manager
      Workload Identity Federation
    Observability
      Firebase Emulators local dev
      Cloud Logging
      Cloud Monitoring
    AI Layer
      pocketsales-agent
        LLM autonomous loop
        Fisher stability scoring
        Risk dataset generation
```

---

## 🔗 Project Repository

**GitHub**: `MasterJavis/android-sales`  
**Version**: `1.6.0` (production-ready)  
**Docs**: 27+ technical documents covering architecture, security, testing, deployment, and data contracts.

---

*Document generated: February 2026*

# PocketSales — Diagramas de Arquitectura

**Última actualización**: 2025-12-28  
**Versión**: 2.0 (Event-Driven con Read Models)

---

## 📊 Comparación: Con vs Sin Read Models

### ❌ Arquitectura SIN Read Models (Problemática)

```mermaid
graph TB
    subgraph "Cliente"
        FE[📱 Frontend<br/>React Native + Expo<br/>Port 8081]
    end
    
    subgraph "Backend API"
        API[⚡ FastAPI<br/>Cloud Run<br/>Port 8082]
    end
    
    subgraph "Firestore - Write Models"
        MI[📄 menu_items<br/>54 docs]
        SI[📦 stock_items<br/>41 docs]
        IS[📊 inventory_stocks<br/>41 docs]
        MIC[🔗 menu_item_components<br/>202 docs]
    end
    
    FE -->|1. GET /pos/availability| API
    
    API -->|2. Query menu_items| MI
    API -->|3. Query stock_items| SI
    API -->|4. Query inventory_stocks| IS
    API -->|5. Query components| MIC
    
    API -->|6. JOIN en memoria<br/>202 componentes<br/>❌ Lento: 2-5s| API
    API -->|7. Calcular disponibilidad<br/>54 items × N ingredientes<br/>❌ CPU intensivo| API
    
    API -->|8. Response<br/>❌ Alta latencia| FE
    
    style API fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style FE fill:#4dabf7,stroke:#1971c2,color:#fff
    style MI fill:#ffd43b,stroke:#f59f00,color:#000
    style SI fill:#ffd43b,stroke:#f59f00,color:#000
    style IS fill:#ffd43b,stroke:#f59f00,color:#000
    style MIC fill:#ffd43b,stroke:#f59f00,color:#000
```

**Problemas:**
- ❌ **4 queries** por cada request
- ❌ **JOINs en memoria** (202 componentes)
- ❌ **Cálculos en runtime** (54 items)
- ❌ **Latencia alta**: 2-5 segundos
- ❌ **Carga en backend**: CPU + memoria
- ❌ **No escalable**: Crece con datos

---

## ✅ Arquitectura CON Read Models (Actual)

### Flujo Completo: Write → Event → Read

```mermaid
graph TB
    subgraph "Cliente"
        FE[📱 Frontend<br/>React Native + Expo<br/>localhost:8081]
    end
    
    subgraph "Backend API - Cloud Run"
        API[⚡ FastAPI<br/>Python 3.12<br/>Port 8082]
        CMD[📝 Command Handler<br/>Write Operations]
        QRY[📖 Query Handler<br/>Read Operations]
    end
    
    subgraph "Event Bus"
        PS[☁️ Pub/Sub<br/>Topic: events<br/>Port 8085 local]
    end
    
    subgraph "Event Processing"
        RMP[🔄 Read Model Projector<br/>Cloud Function Gen2<br/>Python 3.12<br/>512MB RAM]
        DISP[🎯 Dispatcher<br/>Event Router]
        H1[📦 inventory_items.py]
        H2[🍔 pos_availability.py]
        H3[💰 transactions.py]
    end
    
    subgraph "Firestore - Write Models"
        direction LR
        WM1[menu_items]
        WM2[stock_items]
        WM3[inventory_stocks]
        WM4[menu_item_components]
    end
    
    subgraph "Firestore - Read Models"
        direction LR
        RM1[✨ rm_inventory_items<br/>41 docs<br/>Denormalizado]
        RM2[✨ rm_pos_availability<br/>54 docs<br/>Pre-calculado]
        RM3[✨ rm_tx_headers<br/>Transacciones]
        IDMP[🔒 rm_events_processed<br/>Idempotencia]
    end
    
    %% Write Flow
    FE -->|1. POST /inventory/items<br/>Create Item| API
    API --> CMD
    CMD -->|2. Write| WM2
    CMD -->|3. Publish Event<br/>stock_item.created| PS
    API -->|4. 201 Created<br/>✅ Rápido: ~100ms| FE
    
    %% Event Processing
    PS -->|5. Trigger| RMP
    RMP --> DISP
    DISP -->|6. Route by type| H1
    DISP --> H2
    DISP --> H3
    
    H1 -->|7. Check| IDMP
    H1 -->|8. Update| RM1
    H1 -->|9. Mark processed| IDMP
    
    H2 -->|Query BOM| WM4
    H2 -->|Query Stock| WM3
    H2 -->|Update| RM2
    
    %% Read Flow
    FE -.->|10. GET /pos/availability<br/>Query Data| API
    API --> QRY
    QRY -.->|11. Single Query<br/>✅ 1 colección| RM2
    RM2 -.->|12. Response<br/>✅ Rápido: ~50ms| QRY
    QRY -.->|13. JSON| FE
    
    style FE fill:#4dabf7,stroke:#1971c2,color:#fff
    style API fill:#51cf66,stroke:#2f9e44,color:#fff
    style CMD fill:#ff8787,stroke:#c92a2a,color:#fff
    style QRY fill:#69db7c,stroke:#37b24d,color:#fff
    style PS fill:#ffa94d,stroke:#fd7e14,color:#fff
    style RMP fill:#845ef7,stroke:#5f3dc4,color:#fff
    style DISP fill:#9775fa,stroke:#7048e8,color:#fff
    style H1 fill:#da77f2,stroke:#ae3ec9,color:#fff
    style H2 fill:#da77f2,stroke:#ae3ec9,color:#fff
    style H3 fill:#da77f2,stroke:#ae3ec9,color:#fff
    style RM1 fill:#20c997,stroke:#0ca678,color:#fff
    style RM2 fill:#20c997,stroke:#0ca678,color:#fff
    style RM3 fill:#20c997,stroke:#0ca678,color:#fff
    style IDMP fill:#fab005,stroke:#f59f00,color:#000
```

**Beneficios:**
- ✅ **1 query** por request (vs 4)
- ✅ **Sin JOINs** (datos denormalizados)
- ✅ **Sin cálculos** (pre-calculados)
- ✅ **Latencia baja**: ~50ms (vs 2-5s)
- ✅ **Backend ligero**: Solo queries
- ✅ **Escalable**: Async processing

---

## 🔄 Flujo Detallado: Crear Ítem de Inventario

```mermaid
sequenceDiagram
    participant U as 👤 Usuario
    participant FE as 📱 Frontend<br/>(Expo)
    participant API as ⚡ Backend<br/>(FastAPI)
    participant FS as 🗄️ Firestore<br/>(Write Models)
    participant PS as ☁️ Pub/Sub<br/>(events)
    participant RMP as 🔄 Projector<br/>(Cloud Function)
    participant RM as ✨ Firestore<br/>(Read Models)
    
    rect rgb(240, 248, 255)
        Note over U,FS: WRITE FLOW (Síncrono)
        U->>FE: Crear "Tomates, 10kg"
        FE->>API: POST /inventory/items<br/>{name, qty, cost}
        
        Note over API: Command Handler
        API->>FS: Write stock_items
        API->>FS: Write inventory_stocks
        API->>FS: Write stock_item_costs
        
        API->>PS: Publish Event<br/>stock_item.created<br/>{event_id, payload}
        API-->>FE: 201 Created<br/>{stock_item_id}
        FE-->>U: ✅ "Ítem creado"
        Note over U,FE: ⏱️ ~100-200ms
    end
    
    rect rgb(255, 250, 240)
        Note over PS,RM: EVENT PROCESSING (Asíncrono)
        PS->>RMP: Trigger Function<br/>(Pub/Sub push)
        
        Note over RMP: Dispatcher
        RMP->>RMP: Parse envelope
        RMP->>RMP: Route to handler
        
        Note over RMP: inventory_items.py
        RMP->>RM: Check rm_events_processed<br/>(idempotencia)
        
        alt Not processed
            RMP->>FS: Read stock_item
            RMP->>FS: Read inventory_stock
            RMP->>FS: Read stock_item_cost
            
            RMP->>RM: Write rm_inventory_items<br/>(denormalizado)
            RMP->>RM: Write rm_events_processed<br/>(marker)
            
            Note over RMP: pos_availability.py
            RMP->>FS: Query menu_item_components<br/>(BOM)
            RMP->>RM: Query rm_inventory_items<br/>(stock)
            RMP->>RMP: Calculate max_producible<br/>for affected items
            RMP->>RM: Update rm_pos_availability<br/>(54 items)
        else Already processed
            RMP->>RMP: Skip (idempotent)
        end
        
        Note over PS,RM: ⏱️ ~1-3s (async)
    end
    
    rect rgb(240, 255, 240)
        Note over U,RM: READ FLOW (Optimizado)
        U->>FE: Ver inventario
        FE->>API: GET /inventory/summary
        
        Note over API: Query Handler
        API->>RM: SELECT * FROM rm_inventory_items<br/>WHERE business_id = X<br/>AND store_id = Y
        RM-->>API: 41 items (denormalizados)
        API-->>FE: JSON response
        FE-->>U: 📊 Lista de inventario
        Note over U,FE: ⏱️ ~50ms
    end
```

---

## 🏗️ Stack Tecnológico Completo

```mermaid
graph TB
    subgraph "Frontend Layer"
        RN[React Native 0.73<br/>Cross-platform UI]
        EXPO[Expo SDK 50<br/>Dev tooling]
        TS[TypeScript 5.3<br/>Type safety]
    end
    
    subgraph "API Layer - Cloud Run"
        FAST[FastAPI 0.109<br/>Python web framework]
        PYD[Pydantic 2.5<br/>Data validation]
        UVI[Uvicorn<br/>ASGI server]
    end
    
    subgraph "Event Infrastructure"
        PUBSUB[Google Pub/Sub<br/>Message queue<br/>At-least-once delivery]
    end
    
    subgraph "Compute - Cloud Functions Gen2"
        CF[Python 3.12 Runtime<br/>512MB RAM<br/>120s timeout]
        HANDLERS[Event Handlers<br/>inventory_items.py<br/>pos_availability.py<br/>transactions.py]
    end
    
    subgraph "Data Layer - Firestore"
        WRITE[Write Models<br/>Normalized<br/>ACID transactions]
        READ[Read Models<br/>Denormalized<br/>Query-optimized]
    end
    
    subgraph "Development Tools"
        EMU[Firebase Emulators<br/>Local development]
        FIRE[Firebase CLI<br/>Deployment]
    end
    
    RN --> EXPO
    EXPO --> TS
    TS --> FAST
    
    FAST --> PYD
    FAST --> UVI
    FAST --> WRITE
    FAST --> READ
    FAST --> PUBSUB
    
    PUBSUB --> CF
    CF --> HANDLERS
    HANDLERS --> WRITE
    HANDLERS --> READ
    
    EMU -.->|Local| WRITE
    EMU -.->|Local| READ
    EMU -.->|Local| PUBSUB
    
    FIRE -.->|Deploy| CF
    FIRE -.->|Deploy| EMU
    
    style RN fill:#61dafb,stroke:#20232a,color:#000
    style EXPO fill:#000020,stroke:#4630eb,color:#fff
    style TS fill:#3178c6,stroke:#235a97,color:#fff
    style FAST fill:#009688,stroke:#00695c,color:#fff
    style PYD fill:#e92063,stroke:#c2185b,color:#fff
    style PUBSUB fill:#ffa000,stroke:#f57c00,color:#fff
    style CF fill:#4285f4,stroke:#1967d2,color:#fff
    style WRITE fill:#ffd54f,stroke:#fbc02d,color:#000
    style READ fill:#66bb6a,stroke:#43a047,color:#fff
    style EMU fill:#ffca28,stroke:#ffa000,color:#000
```

---

## 📈 Comparación de Performance

```mermaid
graph LR
    subgraph "Sin Read Models"
        Q1[Query 1<br/>menu_items<br/>~50ms]
        Q2[Query 2<br/>stock_items<br/>~50ms]
        Q3[Query 3<br/>inventory_stocks<br/>~50ms]
        Q4[Query 4<br/>components<br/>~100ms]
        JOIN[JOIN en memoria<br/>202 items<br/>~500ms]
        CALC[Calcular disponibilidad<br/>54 items<br/>~1000ms]
        
        Q1 --> Q2
        Q2 --> Q3
        Q3 --> Q4
        Q4 --> JOIN
        JOIN --> CALC
        
        CALC --> RESP1[Total: 2-5s ❌]
    end
    
    subgraph "Con Read Models"
        QRM[Query rm_pos_availability<br/>1 colección<br/>~50ms]
        
        QRM --> RESP2[Total: ~50ms ✅]
    end
    
    style Q1 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style Q2 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style Q3 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style Q4 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style JOIN fill:#ff4757,stroke:#c92a2a,color:#fff
    style CALC fill:#ff4757,stroke:#c92a2a,color:#fff
    style RESP1 fill:#ff4757,stroke:#c92a2a,color:#fff
    
    style QRM fill:#51cf66,stroke:#2f9e44,color:#fff
    style RESP2 fill:#51cf66,stroke:#2f9e44,color:#fff
```

**Mejora**: **40-100x más rápido** 🚀

---

## 🔒 Garantías de Consistencia

```mermaid
stateDiagram-v2
    [*] --> EventPublished: Backend publica evento
    
    EventPublished --> Delivered: Pub/Sub entrega
    
    state "Read Model Projector" as RMP {
        Delivered --> CheckIdempotency: Recibe evento
        CheckIdempotency --> AlreadyProcessed: event_id existe
        CheckIdempotency --> Process: event_id nuevo
        
        Process --> ReadWriteModels: Query source data
        ReadWriteModels --> UpdateReadModel: Calculate
        UpdateReadModel --> MarkProcessed: Write rm_*
        MarkProcessed --> [*]: Commit
        
        AlreadyProcessed --> [*]: Skip (safe)
    }
    
    note right of CheckIdempotency
        rm_events_processed
        doc_id: handler__event_id
        Garantiza idempotencia
    end note
    
    note right of UpdateReadModel
        Eventual Consistency
        Típico: < 1 segundo
        Máximo: < 5 segundos
    end note
```

**Garantías:**
- ✅ **At-least-once delivery** (Pub/Sub)
- ✅ **Idempotencia** (rm_events_processed)
- ✅ **Eventual consistency** (< 1s típico)
- ✅ **No duplicados** (event_id único)

---

## 🎯 Patrones de Diseño Aplicados

```mermaid
mindmap
  root((PocketSales<br/>Architecture))
    CQRS
      Write Models
        Normalized
        Transactional
      Read Models
        Denormalized
        Query-optimized
    Event-Driven
      Pub/Sub
        Decoupling
        Async
      Event Sourcing
        Audit trail
        Replay capability
    Idempotency
      rm_events_processed
        event_id tracking
        Safe retries
    Scalability
      Cloud Functions
        Auto-scaling
        Pay-per-use
      Firestore
        Horizontal scaling
        Sub-10ms reads
```

---

## 📊 Read Models Actuales

| Read Model | Docs | Propósito | Latencia |
|-----------|------|-----------|----------|
| `rm_inventory_items` | 41 | Inventario denormalizado | ~30ms |
| `rm_pos_availability` | 54 | Disponibilidad POS | ~50ms |
| `rm_menu_item_bom` | 202 | BOM forward index | ~40ms |
| `rm_bom_index` | 202 | BOM reverse index | ~40ms |
| `rm_tx_headers` | N | Headers de transacciones | ~30ms |
| `rm_tx_lines` | N | Líneas de transacciones | ~30ms |

**Total**: 6 read models activos

---

## 🚀 Ventajas de la Arquitectura Actual

### Performance
- ✅ **40-100x más rápido** en queries
- ✅ **Latencia predecible** (~50ms)
- ✅ **Escalabilidad horizontal** automática

### Mantenibilidad
- ✅ **Separación de concerns** (CQRS)
- ✅ **Fácil agregar nuevos read models**
- ✅ **Testeable** (event-driven tests)

### Confiabilidad
- ✅ **Idempotencia** garantizada
- ✅ **Retry automático** (Pub/Sub)
- ✅ **Audit trail** completo

### Costos
- ✅ **Pay-per-use** (Cloud Functions)
- ✅ **Scale-to-zero** cuando idle
- ✅ **Menos CPU** en backend

---

**Versión**: 2.0  
**Última actualización**: 2025-12-28  
**Mantenedor**: Equipo PocketSales

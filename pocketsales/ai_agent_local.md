# PocketSales — Agente de IA Local (ps-agent)

> Sistema de generación autónoma de código para producción, integrado con Claude Code via MCP.

---

## ¿Qué es?

`ps-agent` es un **agente de programación autónomo** que corre completamente en hardware local. Recibe tareas de desarrollo en lenguaje natural, genera código de producción, lo valida con un pipeline de calidad multi-etapa, lo testea contra el CI real de `android-sales`, y lo aplica al repositorio.

No es un copilot pasivo — es un agente que ejecuta el ciclo completo: contexto → plan → código → evaluación → aplicación → datos de entrenamiento.

---

## Integración con Claude Code (MCP)

El agente expone un servidor MCP (Model Context Protocol) que Claude Code puede invocar directamente desde el editor:

```
Claude Code (IDE)
    │
    │  mcp__ps-agent__run_task(description="...", tier="pro")
    ▼
ps-agent MCP Server  ←── Python, corre en GPU local
    │
    └──► Orchestrator Pipeline (ver abajo)
```

Desde el editor, una sola llamada lanza el pipeline completo. El resultado incluye `run_id`, status, y artefactos (diff, scores, log de CI).

---

## Pipeline de ejecución

```
TAREA EN LENGUAJE NATURAL
         │
    ┌────▼────┐
    │  Scout  │  qwen2.5-coder:14b — RAG sobre 210K+ vectores del codebase
    └────┬────┘  Identifica el archivo target, recupera contexto relevante
         │
    ┌────▼────┐
    │ Planner │  qwen2.5-coder:14b — genera plan de ejecución (JSON)
    └────┬────┘  Determina archivos a tocar, orden, propósito de cada uno
         │
    ┌────▼────┐
    │Executor │  qwen2.5-coder:32b (GPU) — genera el código completo
    └────┬────┘  Usa prompt especializado: be_worker.yaml / fe_worker.yaml
         │
    ┌────▼────┐
    │Evaluator│  qwen2.5-coder:14b — Judge LLM multi-sample
    └────┬────┘  Puntúa el código contra invariantes y checks estructurales
         │
    ┌────▼────┐
    │ Gate A  │  pytest -m unit/contract sobre android-sales real
    └────┬────┘  Aplica el parche en worktree aislado, corre CI local
         │
    ┌────▼────┐
    │ Commit  │  Si Sf ≥ 0.70 → parche aplicado al repo
    └─────────┘
```

---

## Sistema de scoring (Fisher Score)

Antes de aplicar cualquier parche, el agente calcula un **Fisher Score (Sf)** compuesto:

$$S_f = 0.50 \cdot C_n + 0.20 \cdot (1 - V_n) + 0.30 \cdot Q_n$$

| Componente | Fuente | Qué mide |
|---|---|---|
| **Cn** (Correctness) | LLM Judge, N muestras | ¿El código satisface los invariantes del negocio? |
| **Vn** (Variance) | Similitud Levenshtein entre shadow runs | ¿El modelo produce resultados estables? |
| **Qn** (Quality) | Análisis estático (ast.parse + checks) | Sintaxis válida, patrones arquitecturales correctos |

**Umbral:** `Sf ≥ 0.70` para aplicar el parche. Si no lo alcanza, el agente reintenta con feedback del Judge.

### Incertidumbre Epistémica (Un)

Antes de ejecutar, el agente mide si "sabe lo que hace" — genera N explicaciones técnicas de la tarea con temperatura > 0 y mide la divergencia semántica entre ellas en el espacio de embeddings:

$$U_n = \sigma\!\left(\frac{\bar{d} - 0.18}{0.06}\right), \quad \bar{d} = 1 - \frac{1}{N}\sum_i \cos(\hat{\mathbf{e}}_i,\, \mathbf{c})$$

Si `Un > 0.90` y el plan es ambiguo, el pipeline solicita clarificación en lugar de generar código posiblemente equivocado.

---

## Stack del agente

| Servicio | Tecnología | Nota |
|---|---|---|
| LLM Executor | qwen2.5-coder:32b (GGUF Q4_K_M) | GPU local — 19.8 GB |
| LLM Scout/Planner | qwen2.5-coder:14b | GPU local — 8.9 GB |
| Embeddings | nomic-embed-text (768-dim) | GPU local |
| Vector Store | Qdrant | 210K+ vectores del codebase |
| Serving | Ollama (Docker) | Flash Attention 3 |
| Orquestación | Python async + Claude Code MCP | Servidor JSON-RPC 2.0 |

**Hardware:** NVIDIA RTX 5090 (31.84 GB VRAM) — Blackwell, Flash Attention 3.

---

## Privacidad y seguridad

- **100% local** — ningún fragmento de código sale a internet durante la generación
- El agente accede únicamente al repositorio `android-sales` dentro de rutas autorizadas (`backend/app/`, `android_app/`, `data_pipeline/`, `frontend/`)
- Rutas protegidas (`src/ps_agent/`, `config/`, `.env`) bloqueadas por guard de zona
- Gate A corre en un worktree Git aislado — si los tests fallan, el repo principal no se toca

---

## Flywheel de auto-mejora

Cada run exitoso (Sf ≥ 0.70, Cn ≥ 0.85) genera datos para fine-tuning:

```
Run exitoso
    │
    ├──► datasets/sft/positive_examples_*.jsonl    ← Supervised Fine-Tuning
    ├──► datasets/dpo/preference_pairs_*.jsonl     ← Direct Preference Optimization
    └──► datasets/risk_features/YYYY-MM-DD.jsonl   ← Features para el bandit (LinUCB)
```

A medida que el agente resuelve más tareas del codebase, los datasets crecen y el modelo puede ser fine-tuned para mejorar Cn y Sf en tareas similares futuras.

---

## Tiers de ejecución

| Tier | Modelo Executor | Intentos | Uso |
|---|---|---|---|
| `basic` | qwen2.5-coder:14b | 1–2 | Tareas simples / warmup |
| `pro` | qwen2.5-coder:32b | 2–3 | Tareas de producción (default) |
| `enterprise` | qwen2.5-coder:32b + más thoughts | 3–5 | Tareas complejas multi-archivo |

---

## Ejemplo de uso real

```
Tarea: "Implementar handler OutboxWriter para eventos de stock bajo.
        Target: backend/app/events/low_stock_alert_event.py
        — LowStockAlertEvent dataclass con campos tenant_id, product_id, sku,
          current_stock, reorder_threshold, warehouse_id, timestamp
        — LowStockAlertHandler con db: firestore.Client inyectado
        — Idempotency key: {tenant_id}:{product_id}:{timestamp.isoformat()}"

Resultado:
  run_id  : 20260511_133657_713883
  status  : SUCCESS
  Sf      : 1.0
  Cn      : 1.0
  Gate A  : PASSED (pytest -m unit)
  Tiempo  : ~4 minutos
```

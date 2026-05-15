# PocketSales — Resumen Ejecutivo del Proyecto

> Documento preparado para reunión con Redmol — Mayo 2026

---

## ¿Qué es PocketSales?

PocketSales es un **sistema de punto de venta móvil** (mPOS) multi-tenant diseñado para pequeñas y medianas empresas en Latinoamérica. Permite a negocios gestionar inventario, ventas, recetas (BOM), turnos y sincronización offline desde un dispositivo Android.

---

## Stack tecnológico

| Capa | Tecnología | Rol |
|---|---|---|
| Frontend | React Native + Expo (TypeScript) | App Android multi-tienda |
| Backend API | FastAPI (Python 3.12) en Cloud Run | REST API multi-tenant |
| Base de datos | Google Firestore | Write models + Read models |
| Eventos | Google Pub/Sub | Bus de eventos async |
| Proyectores | Cloud Functions Gen2 (Python) | Read model builders |
| Auth | Firebase Auth + RBAC custom | Autenticación por negocio/tienda |
| Infra | Terraform + Cloud Build | IaC + CI/CD en GCP |

---

## Arquitectura: CQRS + Event-Driven

El sistema separa lecturas de escrituras (CQRS) con un bus de eventos asíncrono:

```
[App móvil]
    │
    ▼ POST /inventory/items (~100ms)
[FastAPI / Cloud Run]
    │ 1. Escribe write model (Firestore)
    │ 2. Publica evento (Pub/Sub)
    │
    ▼ (async ~1s)
[Cloud Function — Projector]
    │ Procesa evento con idempotencia
    │
    ▼
[Read Models — Firestore rm_*]
    │
    ▼ GET /pos/availability (~50ms)
[App móvil]
```

**Resultado:** lecturas 40–100× más rápidas que un esquema JOIN tradicional.

---

## Estado actual del producto

| Módulo | Estado |
|---|---|
| Inventario multi-tenant | Producción |
| Punto de venta (POS) | Producción |
| Recetas / BOM | Producción |
| Sincronización offline (SQLite) | En desarrollo |
| Turnos y reportes | En desarrollo |
| Pagos integrados | Roadmap |

---

## Diferenciador técnico: Agente de IA local

PocketSales cuenta con un **agente de programación autónomo** que corre completamente en hardware local (GPU dedicada) y opera sobre el codebase via Claude Code. Ver documento `ai_agent_local.md` para detalle completo.

En síntesis:
- El agente recibe tareas en lenguaje natural y genera, valida y aplica código de producción de forma autónoma
- Corre modelos `qwen2.5-coder:32b` en GPU local — sin enviar código a la nube
- Incluye pipeline de control de calidad con scoring matemático (Fisher Score) antes de aplicar cualquier parche
- Los datos generados (ejemplos positivos, pares de preferencia) se acumulan para fine-tuning futuro del modelo

---

## Repositorios

| Repo | Descripción |
|---|---|
| `android-sales` | Codebase principal: app + backend + data pipeline |
| `pocketsales-agent` | Agente de IA autónomo que desarrolla sobre `android-sales` |

---

## Contacto

**Javier** — Fundador & CTO  
Reunión Redmol: Martes 19, 5:00pm (Google Meet)

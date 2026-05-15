# Documentos recomendados para compartir con Simon (Redmol)

Reunión: Martes 19, 5:00pm

---

## ✅ Seguros para compartir directamente

### De este folder (`personal_projects/pocketsales/`)

| Archivo | Qué muestra | Recomendación |
|---|---|---|
| `README.md` | Overview ejecutivo del proyecto completo | ⭐ Empezar aquí |
| `architecture.md` | Diagramas CQRS + event-driven con Mermaid | ⭐ Impacto técnico visual |
| `ai_agent_local.md` | El agente autónomo de IA + scoring matemático | ⭐ Diferenciador único |

### Del repo `android-sales` (origen)
- `docs/architecture_diagrams.md` — mismo que `architecture.md` arriba
- `docs/local_development_guide.md` — stack de desarrollo local
- `docs/testing/unit_tests_guide.md` — cultura de calidad de código

### Del repo `pocketsales-agent` (origen)
- `docs/AGENT_DEVELOPMENT_GUIDE.md` — guía técnica del agente
- `docs/fisher_scoring.md` — spec matemática del scoring system

---

## ⚠️ NO compartir (info sensible o interna)

| Archivo | Por qué no |
|---|---|
| `PS_AGENT_MCP.md` (Desktop) | Tiene paths locales de Windows, config interna |
| `infra/terraform/IAM_PERMISSIONS.md` | Permisos GCP de producción |
| `backend/PRODUCTION_SEEDING.md` | Datos de seeding productivo |
| `docs/security-guide.md` | Vectores de ataque internos |
| `config/android_sales_architecture.md` | Contrato interno del agente, no es docu externa |
| `.env`, `credentials/` | Obvio |

---

## Sugerencia de orden para la reunión

1. **README.md** — 2 min: qué es PocketSales, mercado objetivo
2. **architecture.md** — 5 min: los diagramas Mermaid hablan solos (CQRS, event-driven, latencias)
3. **ai_agent_local.md** — 5 min: el agente autónomo como diferenciador técnico
4. **Preguntas** — 33 min restantes

El agente es el diferenciador más fuerte para una conversación con un CEO técnico — es el único proyecto de este tipo corriendo completamente local sobre hardware propio en la región.

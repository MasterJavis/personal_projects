# ps-agent — Código Fuente: Sistema Multi-Agente

> Código real de los módulos principales del pipeline de IA autónomo.
> Sin rutas locales. Extraído directamente de `pocketsales-agent`.

---

## 1. Orchestrator (`orchestrator/main.py`)

Coordina el pipeline completo. El `Orchestrator` es un singleton con pre-warm en thread daemon
para eliminar el cold-start del primer `run_task`.

### Imports y constantes

```python
from datetime import datetime
import asyncio
import re
import sys
import yaml
from rich.console import Console
from pathlib import Path

from ps_agent.structs.messages import AgentTask, TaskStatus, WorkerRole
from ps_agent.structs.evidence import EvidenceBundle
from ps_agent.workers.factory import create_worker
from ps_agent.config import settings
from ps_agent.structs.trace import ExecutionTrace, TaskInfo, ModelInfo
from ps_agent.workers.shadow_executor import ShadowExecutor
from ps_agent.guards.fisher_scoring import FisherScorer
from ps_agent.llm.ollama import OllamaClient
from ps_agent.llm.registry import ModelRegistry
from ps_agent.guards.uncertainty import UncertaintyScorer
from ps_agent.monitors.risk_monitor import RiskMonitor
from ps_agent.router.difficulty_router import DifficultyRouter
from ps_agent.risk.feature_store import RiskFeatureStore
from ps_agent.rl.linucb import LinUCBBandit
from ps_agent.rl.context import build_context
from ps_agent.rl.temperature_policy import TemperaturePolicy
from ps_agent.workers.planner import PlannerWorker
from ps_agent.guards.domain_router import DomainRouter
from ps_agent.evaluator.plan_validator import PlanValidator
from ps_agent.analysis.impact_analyzer import ImpactAnalyzer
from ps_agent.economics.emv_policy import EMVPolicyEngine
from ps_agent.economics.loader import load_champion_models, load_cost_config
from ps_agent.llmops.sft_logger import SFTLogger
from ps_agent.guards.code_quality import compute_qn_detailed
from ps_agent.gates.gate_a import GateARunner
from ps_agent.bandit.linucb import LinUCB as GenerationBandit

console = Console()

# SFT quality guard — rechaza código placeholder antes de persistir
_VACUOUS_SFT_PATTERNS = (
    "assert True", "pass  # Placeholder", "pass  # placeholder",
    "pass  # TODO", "pass  # todo", "raise NotImplementedError",
    "def test_example_function():", "your_function(",
)

def _is_vacuous_sft(code: str) -> bool:
    if not code:
        return True
    for pattern in _VACUOUS_SFT_PATTERNS:
        if pattern in code:
            return True
    return False
```

### Clase Orchestrator — `__init__`

```python
class Orchestrator:
    def __init__(self):
        self.git_tools = GitTools(settings.REPO_ROOT)
        self.run_id = datetime.now().strftime("%Y%m%d_%H%M%S")
        self.evidence_dir = settings.ARTIFACTS_DIR / "runs" / self.run_id
        self.bundle = None
        self.fisher_config = self._load_fisher_config()
        self._load_commercial_profile()
        self.active_llm_client = None

        # Carga hardened: local model must load or crash (no fallbacks)
        self.active_llm_client = self._load_hardened_local_model()

        # Mixture of Agents: enruta cada rol al tier correcto de modelo
        self.registry = self._build_model_registry()

        # Online Feature Store para el data flywheel
        self.feature_store = RiskFeatureStore()

        # LinUCB Bandit — selecciona thoughts, attempts, temperature
        _bandit_cfg = self._load_bandit_config()
        self.bandit = LinUCBBandit(
            arms=_bandit_cfg["arms"],
            alpha=_bandit_cfg["alpha"]
        )
        self.temperature_policy = TemperaturePolicy()
        self.reward_buffer = []

        # Generation Strategy Bandit (arm-level: T / ctx / prefix)
        _gen_state_path = settings.REPO_ROOT / "data" / "bandit" / "linucb_state.npz"
        self.generation_bandit = GenerationBandit(
            alpha=0.5, state_path=_gen_state_path, load_existing=True
        )

        # RiskMonitor (pasivo, observabilidad)
        if settings.RISK_MONITOR_ENABLED:
            self.risk_monitor = RiskMonitor(bundle_path=settings.RISKGATE_MODEL_PATH)
        else:
            self.risk_monitor = None

        # EMV Policy Engine — decisión económica sobre intentos
        models_dict = load_champion_models(alias="Production")
        c_retry_df, c_future_loss_df = load_cost_config()
        self.emv_engine = EMVPolicyEngine(
            champion_models=models_dict,
            C_retry=c_retry_df,
            C_future_loss=c_future_loss_df,
            feature_cols=["attempt_index", "un_score", "geo_score", "vn_score"],
            model_alias="Production"
        )

        # Gate A — CI bridge hacia android-sales
        self.gate_a_runner = GateARunner()
```

### `_build_model_registry` — Mixture of Agents

```python
def _build_model_registry(self) -> ModelRegistry:
    """
    Enruta cada rol al tier correcto:
      heavy (32B) → executor
      medium (14B) → evaluator
      light  (14B) → scout, planner, uncertainty, fisher shadow runs
    """
    if not settings.ENABLE_MODEL_ROUTING:
        return ModelRegistry.build(heavy_client=self.active_llm_client)

    from ps_agent.llm.registry import ROLE_CTX
    light  = OllamaClient(model=settings.LIGHT_MODEL,  num_ctx=ROLE_CTX["scout"])
    medium = OllamaClient(model=settings.MEDIUM_MODEL, num_ctx=ROLE_CTX["evaluator"])

    return ModelRegistry.build(
        heavy_client=self.active_llm_client,
        light_client=light,
        medium_client=medium,
    )
```

### `measure_uncertainty` — Incertidumbre Epistémica

```python
async def measure_uncertainty(self, task_description: str) -> float:
    """Self-Consistency: genera N explicaciones técnicas y mide su divergencia semántica."""
    uncertainty_client = self.registry.for_role("uncertainty")
    scorer = UncertaintyScorer(uncertainty_client)

    prompt = (
        f"Context: {task_description}\n\n"
        "Explain the core technical logic needed to implement this in exactly one short sentence. "
        "Do not write code."
    )

    plans = []
    console.print(f"   [dim]Generating {self.thoughts_count} strategic thoughts...[/dim]")
    for _ in range(self.thoughts_count):
        resp = await uncertainty_client.generate(prompt, temperature=self.base_temperature)
        text = resp.text if hasattr(resp, 'text') else str(resp)
        plans.append(text)

    return await scorer.compute_un(plans)
```

### `run_pipeline` — Flujo del pipeline

```python
async def run_pipeline(self, task_description: str, tier: str = "pro"):
    import time as _time
    _wall_start = _time.monotonic()

    if not self.bundle or self.bundle.task_description != task_description:
        self.start_new_run(task_description)

    trace = ExecutionTrace(
        run_id=self.run_id, environment="local",
        task=TaskInfo(issue_id=f"ISSUE-{self.run_id}", task_type="feature"),
        model=ModelInfo(model_id=settings.LLM_MODEL,
                        temperature=self.base_temperature,
                        num_ctx=settings.NUM_CTX)
    )

    # === PATH SANITIZER ===
    # Strip Windows absolute paths embedded en la descripción de tarea.
    # Convierte: C:\...\android-sales\backend\app\api\v1\sync.py
    #        en: backend/app/api/v1/sync.py
    import re as _re
    _TARGET_ROOTS = [
        settings.TARGET_REPO_PATH.replace("\\", "/"),
        settings.TARGET_REPO_PATH.replace("\\", "\\\\"),
        settings.REPO_ROOT.replace("\\", "/"),
        settings.REPO_ROOT.replace("\\", "\\\\"),
    ]
    _sanitized = task_description
    for _root in _TARGET_ROOTS:
        _abs_pattern = _re.compile(
            _re.escape(_root).replace("/", r"[/\\\\]") + r"[/\\\\]?",
            _re.IGNORECASE
        )
        _sanitized = _abs_pattern.sub("", _sanitized)
    _sanitized = _re.sub(r'(?<=[a-zA-Z0-9_])\\(?=[a-zA-Z0-9_])', '/', _sanitized)
    if _sanitized != task_description:
        console.print("[dim]🔧 Sanitized absolute path(s) from task description.[/dim]")
        task_description = _sanitized

    # === DIFFICULTY ROUTER (Dynamic Budget) ===
    router = DifficultyRouter(llm_client=self.registry.for_role("planner"))
    route_decision = await router.route(task_description, tier=tier)
    v_task = route_decision.get("v_task", 2500)

    un_score = 0.0  # Se calcula después del Planner

    # --- SCOUT ---
    worker_scout = create_worker(WorkerRole.SCOUT, llm_client=self.registry.for_role("scout"))
    task_scout = AgentTask(id="task-scout", description=task_description, role=WorkerRole.SCOUT)
    result_scout = await worker_scout.execute(task_scout)
    self.bundle.add_step("Scout", "Identify Target", task_description, result_scout.notes)

    # --- PLANNER ---
    worker_planner = PlannerWorker(role=WorkerRole.PLANNER, llm_client=self.registry.for_role("planner"))
    scout_hint = (result_scout.notes or "").strip()
    planner_description = task_description
    if scout_hint and "/" in scout_hint:
        planner_description = (
            f"{task_description}\n\n"
            f"SCOUT RECOMMENDATION — primary target file: {scout_hint}\n"
            f"Use the exact path recommended by Scout unless you have strong architectural reasons to deviate."
        )
    task_planner = AgentTask(id="task-planner", description=planner_description, role=WorkerRole.PLANNER)
    result_planner = await worker_planner.execute(task_planner)

    self.bundle.add_step(
        "Planner", "Structure Execution Plan",
        planner_description[:200], result_planner.notes or "(no output)"
    )

    if result_planner.status != TaskStatus.COMPLETED or not result_planner.metadata:
        console.print("[red]⛔ Planner failed to structure the task. Aborting.[/red]")
        self.bundle.final_status = "FAILED"
        return self._return_summary(trace, "FAILED", v_task, 0, un_score, 0, 0, 0)

    plan = result_planner.metadata.get("plan", {})
    execution_order = plan.get("execution_order", [])
    console.print(f"   [dim]Planner execution_order: {execution_order}[/dim]")

    # Guard: empty execution_order → FAILED (no vacuous SUCCESS)
    if not execution_order:
        console.print("[red]⛔ Planner returned empty execution_order. Failing.[/red]")
        self.bundle.final_status = "FAILED"
        return self._return_summary(trace, "FAILED", v_task, 0, un_score, 0, 0, 0)

    # --- EXECUTOR loop (FE_DEV / BE_DEV routing) ---
    _FE_PREFIXES = ("android_app/", "frontend/")
    _FE_EXTENSIONS = (".ts", ".tsx", ".js", ".jsx")

    for current_file in execution_order:
        for attempt in range(self.max_attempts):
            _is_frontend = (
                any(current_file.startswith(p) for p in _FE_PREFIXES)
                or any(current_file.endswith(ext) for ext in _FE_EXTENSIONS)
            )
            _exec_role = WorkerRole.FE_DEV if _is_frontend else WorkerRole.BE_DEV
            worker_exec = create_worker(_exec_role, llm_client=self.registry.for_role("executor"))

            task_exec = AgentTask(
                id=f"exec-{current_file}-v{attempt}",
                description=exec_desc,
                role=_exec_role
            )
            result_exec = await worker_exec.execute(task_exec)

            # EVALUATOR (Judge)
            task_eval = AgentTask(
                id=f"eval-{current_file}-v{attempt}",
                description=task_description,
                role=WorkerRole.EVALUATOR,
                context={"target_file": current_file}
            )
            result_eval = await worker_evaluator.execute(task_eval)
            cn_score = result_eval.metadata.get("consistency_score", 0.0)

            # Fisher Score
            sf = fisher_scorer.compute_sf(un=un_score, vn=vn, c=cn_score, qn=qn)

            # Geometric Guard
            geo_result = await geo_guard.check_adherence(generated_code, current_file)

            # Gate A — CI
            gate_result = await self.gate_a_runner.run(patch=generated_code, target_file=current_file)

            if sf >= 0.70 and gate_result.passed:
                # Commit patch + log SFT positive example
                break
```

---

## 2. Planner Worker (`workers/planner.py`)

```python
import json
import re
try:
    import json_repair
    _JSON_REPAIR_AVAILABLE = True
except ImportError:
    _JSON_REPAIR_AVAILABLE = False
from ps_agent.workers.base import BaseWorker
from ps_agent.structs.messages import AgentTask, TaskResult, TaskStatus

class PlannerWorker(BaseWorker):
    def __init__(self, role, llm_client=None, store_id: str = "pocketsales_default"):
        super().__init__(role, llm_client=llm_client, store_id=store_id)

    async def execute(self, task: AgentTask, **kwargs) -> TaskResult:
        print(f"   [Planner] 🧠 Designing execution plan for: {task.description[:50]}...")

        # Detect single-file override
        _single_file_override = (
            "write EVERYTHING in this single file" in task.description or
            "IMPLEMENTATION NOTES — write EVERYTHING in this single file" in task.description
        )

        prompt = (
            "You are the Lead Software Architect. Your job is to break down the following task "
            "into a multi-file execution plan. Do NOT write the actual code, only the plan.\n\n"
            f"TASK:\n{task.description}\n\n"
            "MANDATORY RULES:\n"
            "1. Output MUST be ONLY a valid JSON object.\n"
            "2. Identify all files that need to be created or modified.\n"
            + (
                "3. SINGLE-FILE OVERRIDE (ABSOLUTE PRIORITY): The task description says to write "
                "EVERYTHING in a single file. Your plan MUST contain exactly ONE file. Do NOT add "
                "separate test files, emitter files, repository files, or any other files.\n"
                if _single_file_override else
                "3. PLAN MINIMALITY: DO NOT over-engineer. DO NOT add test files, routers, schemas, or configs UNLESS:\n"
                "   a) They are EXPLICITLY named in the task description, OR\n"
                "   b) The task says 'create tests', 'add unit tests', 'write tests', or similar.\n"
                "   When the task mentions a 'tests/' path directly, you MUST include it in the plan.\n"
                "   In all other cases, stick strictly to the requested scope.\n"
            ) +
            "4. ARCHITECTURAL INVARIANTS:\n"
            "   - If a schema is modified, you MUST include: (a) event update, (b) projection/read_model update.\n"
            "   - Order MUST respect architectural layering:\n"
            "     1. Schema → 2. Events → 3. Projections → 4. API/Controllers → 5. Tests\n"
            "   - If modifying any event envelope structure, MUST include: 'Increment envelope version'.\n\n"
            "PATH RULES (CRITICAL):\n"
            "1. SCOUT RECOMMENDATION (HIGHEST PRIORITY): If the task description contains "
            "'SCOUT RECOMMENDATION — primary target file:', you MUST use EXACTLY that path "
            "in execution_order. Do NOT substitute or invent an alternative path.\n"
            "2. DUAL-REPO PATHS — all of the following prefixes are VALID and ALLOWED:\n"
            "   - src/ps_agent/tools/        ← agent tools (Python)\n"
            "   - backend/app/               ← android-sales backend (Python/FastAPI)\n"
            "   - backend/tests/             ← android-sales backend tests\n"
            "   - android_app/               ← React Native / Expo frontend (TypeScript)\n"
            "   - data_pipeline/             ← Cloud Functions / projectors\n"
            "   - frontend/                  ← web frontend\n"
            "   - tests/unit/                ← agent unit tests\n"
            "   - tests/integration/         ← agent integration tests\n"
            "3. FORBIDDEN paths: workspace/, ./, or absolute paths (e.g. C:/... or /home/...).\n"
            "4. All paths MUST be strictly relative to repository root.\n\n"
            "EXPECTED JSON FORMAT (Python example):\n"
            "{\n"
            '  "files": [\n'
            '    {\n'
            '      "path": "backend/app/events/low_stock_alert_event.py",\n'
            '      "purpose": "Implement OutboxWriter-based low-stock alert handler",\n'
            '      "dependencies": []\n'
            '    }\n'
            '  ],\n'
            '  "execution_order": ["backend/app/events/low_stock_alert_event.py"]\n'
            "}\n\n"
            "EXPECTED JSON FORMAT (TypeScript example):\n"
            "{\n"
            '  "files": [\n'
            '    {\n'
            '      "path": "android_app/src/lib/session.ts",\n'
            '      "purpose": "Add post-login background sync trigger",\n'
            '      "dependencies": []\n'
            '    }\n'
            '  ],\n'
            '  "execution_order": ["android_app/src/lib/session.ts"]\n'
            "}"
        )

        raw_response = await self.llm.generate(prompt, temperature=0.1)
        text = raw_response.text if hasattr(raw_response, 'text') else str(raw_response)

        # Extracción robusta del JSON
        match = re.search(r'```(?:json)?\s*(.*?)\s*```', text, re.DOTALL | re.IGNORECASE)
        json_str = match.group(1).strip() if match else text[text.find('{'):text.rfind('}')+1]

        try:
            if _JSON_REPAIR_AVAILABLE:
                plan = json_repair.loads(json_str)
            else:
                plan = json.loads(json_str)

            if "execution_order" not in plan or "files" not in plan:
                raise ValueError("JSON missing 'execution_order' or 'files' keys.")

            return TaskResult(
                worker_id=self.role.value, task_id=task.id, status=TaskStatus.COMPLETED,
                notes="Plan generated successfully",
                metadata={"plan": plan, "tokens_used": getattr(raw_response, 'total_tokens', 0)}
            )
        except Exception as e:
            print(f"   [Planner] ❌ Failed to parse plan: {e}. Attempting raw-text fallback...")

            # Fallback: extraer cualquier ruta de archivo del texto
            file_paths = re.findall(r'[\w/\-]+\.(?:py|ts|tsx|js|yaml|md|json|sh)', text)
            unique_paths = list(dict.fromkeys(file_paths))
            if unique_paths:
                fallback_plan = {
                    "files": [{"path": p, "purpose": "Auto-extracted from LLM response",
                               "dependencies": []} for p in unique_paths],
                    "execution_order": unique_paths
                }
                print(f"   [Planner] ⚠️ Using fallback plan with {len(unique_paths)} file(s): {unique_paths}")
                return TaskResult(
                    worker_id=self.role.value, task_id=task.id, status=TaskStatus.COMPLETED,
                    notes="Plan recovered via regex fallback",
                    metadata={"plan": fallback_plan, "tokens_used": getattr(raw_response, 'total_tokens', 0)}
                )

            return TaskResult(
                worker_id=self.role.value, task_id=task.id, status=TaskStatus.FAILED,
                notes=f"JSON Parse Error (json_repair also failed): {e}"
            )
```

---

## 3. Evaluator / Judge (`workers/evaluator.py`)

Judge multi-sample: evalúa Cn (consistencia) con N=2 muestras a T=0.30.
Consenso conservador: si las 2 muestras difieren >0.30, toma el score menor.

```python
import ast
import json
import re
import numpy as np
from pathlib import Path
from typing import List, Optional

from ps_agent.workers.base import BaseWorker
from ps_agent.structs.messages import AgentTask, TaskResult, TaskStatus
from ps_agent.structs.evaluation import JudgeResult
from ps_agent.config import settings

import os as _os
NUM_JUDGE_SAMPLES = int(_os.environ.get("PS_JUDGE_SAMPLES", 2))
JUDGE_TEMP = 0.30
DISAGREEMENT_THRESHOLD = 0.30


class EvaluatorWorker(BaseWorker):

    def _extract_target_from_task(self, description: str) -> Optional[str]:
        """Robustly extract target file (supports src/, workspace/, etc)."""
        if not description:
            return None
        description = description.replace("\\n", "\n")
        for line in description.splitlines():
            clean = line.strip()
            if clean.upper().startswith("TARGET FILES:"):
                raw = clean.split("TARGET FILES:", 1)[1].strip()
                raw = raw.replace("['", "").replace("']", "").replace('["', '').replace('"]', '')
                return raw.split(",")[0].strip()
        match = re.search(r"(?:src|workspace)/[\w/]+\.(?:py|md|txt|yaml|sh|tsx|ts|js|json)", description)
        if match:
            return match.group(0)
        return None

    def _extract_invariants(self, description: str) -> List[str]:
        """
        Extrae MANDATORY INVARIANTS del task description.
        Estrategia (en prioridad):
        1. Bloque explícito === MANDATORY INVARIANTS ... ===
        2. Requisitos numerados (1) … (2) …
        3. Bullet list bajo Requirements:/Constraints:
        """
        pattern = re.compile(
            r"=== MANDATORY INVARIANTS.*?===(.*?)=== END INVARIANTS ===",
            re.DOTALL | re.IGNORECASE,
        )
        match = pattern.search(description)
        if match:
            block = match.group(1).strip()
            invariants = []
            for line in block.splitlines():
                line = re.sub(r"^\d+\.\s*|^[-*]\s*", "", line.strip()).strip()
                if line:
                    invariants.append(line)
            return invariants

        req_section = re.search(
            r"(?:Requisitos\s+estrictos|Requirements?|Constraints?)\s*:(.+?)(?:\.\s*$|\Z)",
            description, re.IGNORECASE | re.DOTALL,
        )
        section_text = req_section.group(1) if req_section else description
        parts = re.split(r"\s*(?:\(\d+\)|\d+\.)\s+", section_text)
        numbered = [p.strip().rstrip(".") for p in parts if len(p.strip()) >= 10]
        if numbered:
            return numbered

        bullet_section = re.search(
            r"(?:Requirements?|Constraints?)\s*:\s*\n((?:\s*[-*•]\s+.+\n?)+)",
            description, re.IGNORECASE,
        )
        if bullet_section:
            bullets = []
            for line in bullet_section.group(1).splitlines():
                item = re.sub(r"^\s*[-*•]\s+", "", line).strip()
                if item:
                    bullets.append(item)
            return bullets

        return []

    def _extract_first_valid_json(self, text: str) -> Optional[dict]:
        for i in range(len(text)):
            if text[i] == "{":
                for j in range(len(text), i, -1):
                    if text[j - 1] == "}":
                        candidate = text[i:j]
                        try:
                            return json.loads(candidate)
                        except json.JSONDecodeError:
                            continue
        return None

    def _build_judge_prompts(
        self,
        task_description: str,
        target_file: Optional[str],
        file_exists: bool,
        code_content: str,
        invariants: List[str],
        is_test_file: bool = False,
    ):
        """Construye system + user prompt para el Judge.

        El branch de structural checks depende del tipo de archivo:
          is_test_file → checks de tests (def test_, mock, assert)
          is_typescript → checks TypeScript/React Native
          else          → checks Python backend (firestore.Client inyectado)
        """
        file_ext = Path(target_file).suffix if target_file else ".py"
        is_python = file_ext == ".py"
        _FE_PREFIXES = ("android_app/", "frontend/")
        _TS_EXTS = (".ts", ".tsx", ".js", ".jsx")
        is_typescript = (
            any((target_file or "").startswith(p) for p in _FE_PREFIXES)
            or file_ext in _TS_EXTS
        )

        if is_typescript:
            lang_name, wrapper = "TypeScript", "typescript"
        elif is_python:
            lang_name, wrapper = "Python", "python"
        else:
            lang_name, wrapper = "Content", "markdown"

        system_prompt = (
            "You are a Deterministic Compliance Judge. "
            f"Your ONLY job is to validate {lang_name} content against requirements.\n"
            "OUTPUT FORMAT: strictly valid JSON matching this schema:\n"
            "{\n"
            '  "reasoning": "REQUIRED — cite specific lines/patterns that justify your score",\n'
            '  "consistency_score": float (0.0-1.0, mean of all sub-scores below),\n'
            '  "invariant_scores": list of floats (one 0.0 or 1.0 per invariant),\n'
            '  "phantom_files": bool,\n'
            '  "missing_acceptance": bool,\n'
            '  "violates_constraints": bool\n'
            "}\n"
            "SCORING METHOD:\n"
            "  1. Score each INVARIANT as 1.0 (satisfied) or 0.0 (violated/missing).\n"
            "  2. Score STRUCTURAL checks as 1.0 or 0.0.\n"
            "  3. consistency_score = mean of ALL individual scores.\n"
            "  Do NOT round to 0.0 or 1.0 — use the exact mean."
        )

        if invariants:
            inv_block_lines = ["MANDATORY INVARIANTS (score each 1.0=satisfied, 0.0=violated):"]
            for idx, inv in enumerate(invariants, 1):
                inv_block_lines.append(f"  {idx}. {inv}")
            inv_block = "\n".join(inv_block_lines)
        else:
            inv_block = "(No explicit invariants defined — score based on general task requirements.)"

        if is_test_file:
            structural_checks = (
                "STRUCTURAL CHECKS — TEST FILE (score each 1.0 or 0.0):\n"
                "  A. Has at least one `def test_` function\n"
                "  B. Uses `unittest.mock` / `MagicMock` for all external dependencies\n"
                "  C. Every test contains at least one assert statement\n"
                "  D. Satisfies the stated purpose/task requirement\n"
            )
        elif is_typescript:
            structural_checks = (
                "STRUCTURAL CHECKS — TYPESCRIPT/REACT NATIVE FILE (score each 1.0 or 0.0):\n"
                "  A. Satisfies the stated purpose/task requirement (correct logic implemented)\n"
                "  B. No forbidden imports: does NOT import from 'ps_agent', 'get_store_db', or Python modules\n"
                "  C. TypeScript types are used where appropriate (interfaces, type annotations)\n"
                "  D. Follows the patterns described in the task (e.g. fire-and-forget IIFE, Zustand store, apiFetch)\n"
                "  NOTE: Python-style classes and get_store_db do NOT apply to TypeScript.\n"
            )
        else:
            structural_checks = (
                "STRUCTURAL CHECKS — PYTHON BACKEND IMPLEMENTATION FILE (score each 1.0 or 0.0):\n"
                "  A. Defines at least one class whose __init__ accepts business_id and store_id as parameters\n"
                "  B. Firestore client (db) is INJECTED as a parameter — NEVER obtained internally via get_store_db()\n"
                "     CORRECT: def __init__(self, db: firestore.Client, business_id: str, store_id: str)\n"
                "     FORBIDDEN: from ps_agent.db.context import get_store_db  ← score 0.0 if present\n"
                "  C. All public methods have type hints on parameters and return value\n"
                "  D. Satisfies the stated purpose/task requirement\n"
                "  NOTE: test functions belong in a SEPARATE test file — do NOT deduct for their absence here.\n"
            )

        user_prompt = (
            f"TASK: {task_description}\n"
            f"TARGET FILE: {target_file} (Exists: {file_exists})\n\n"
            f"{inv_block}\n\n"
            f"{structural_checks}\n"
            f"EVIDENCE ({lang_name} CONTENT):\n"
            f"```{wrapper}\n"
            f"{code_content[:8000]}\n"
            "```\n\n"
            "SCORING RULES:\n"
            "  - If file is missing/empty but required: all scores = 0.0\n"
            "  - Score invariants and structural checks independently\n"
            "  - consistency_score = mean(invariant_scores + structural_scores)\n"
            "\nPROVIDE JSON VERDICT:"
        )

        return system_prompt, user_prompt

    async def execute(self, task: AgentTask) -> TaskResult:
        import asyncio
        print("   [Evaluator] ⚖️  Auditing Consistency (Cn) — Multi-Sample Judge...")

        # 1. Resolver target file
        target_file = task.context.get("target_file")
        if not target_file:
            target_file = self._extract_target_from_task(task.description)

        code_content = "FILE_NOT_FOUND"
        file_exists = False

        if target_file:
            if target_file.startswith("workspace/"):
                target_file = target_file.replace("workspace/", "", 1)

            # Dual-repo routing: android-sales paths → TARGET_REPO_PATH
            _android_prefixes = ("backend/", "android_app/", "data_pipeline/", "frontend/")
            _base_root = (
                Path(settings.TARGET_REPO_PATH)
                if any(target_file.startswith(p) for p in _android_prefixes)
                else Path(settings.REPO_ROOT)
            )
            abs_path = (_base_root / target_file).resolve()

            for attempt in range(3):
                if abs_path.exists():
                    code_content = abs_path.read_text(encoding="utf-8")
                    file_exists = True
                    break
                else:
                    if attempt < 2:
                        await asyncio.sleep(1.0)

        # 2. AST syntax pre-check (fast fail para Python)
        file_ext = Path(target_file).suffix if target_file else ".py"
        is_python = file_ext == ".py"
        if file_exists and code_content.strip() and is_python:
            try:
                ast.parse(code_content)
            except SyntaxError as e:
                fail_result = JudgeResult(
                    consistency_score=0.0, phantom_files=False,
                    missing_acceptance=True, violates_constraints=True,
                    reasoning=f"Code is not valid Python. Syntax Error: {str(e)}",
                    invariant_scores=[], sample_scores=[0.0],
                )
                return TaskResult(
                    worker_id=self.role, task_id=task.id,
                    status=TaskStatus.COMPLETED,
                    notes=json.dumps(fail_result.model_dump()),
                    metadata=fail_result.model_dump()
                )

        # 3. Determinar tipo de archivo
        _tf = str(target_file or "")
        is_test_file: bool = (
            task.context.get("is_test_file", False)
            or _tf.startswith("tests/") or "/test_" in _tf or _tf.startswith("test_")
        )

        # 4. Extraer invariantes
        invariants = self._extract_invariants(task.description)

        # 5. Construir prompts
        system_prompt, user_prompt = self._build_judge_prompts(
            task_description=task.description,
            target_file=target_file, file_exists=file_exists,
            code_content=code_content, invariants=invariants,
            is_test_file=is_test_file,
        )

        # 6. Multi-sample inference
        raw_scores: List[float] = []
        all_invariant_scores: List[List[float]] = []
        last_data: Optional[dict] = None
        text_resp = ""

        for sample_idx in range(NUM_JUDGE_SAMPLES):
            response = await self.llm.generate(
                system=system_prompt, prompt=user_prompt,
                temperature=JUDGE_TEMP, max_tokens=1536,
            )
            text_resp = response.text if hasattr(response, "text") else str(response)
            data = self._extract_first_valid_json(text_resp)

            if data:
                if "score" in data and "consistency_score" not in data:
                    data["consistency_score"] = data.pop("score")
                score = float(data.get("consistency_score", 0.0))
                raw_scores.append(score)
                inv_scores = data.get("invariant_scores", [])
                if isinstance(inv_scores, list):
                    all_invariant_scores.append([float(s) for s in inv_scores])
                last_data = data

        if not raw_scores:
            return TaskResult(
                worker_id=self.role, task_id=task.id, status=TaskStatus.FAILED,
                notes="Evaluator Error: No valid JSON found in any judge sample.",
                metadata={"raw_output": text_resp}
            )

        # 7. Consenso conservador
        if len(raw_scores) == 2 and abs(raw_scores[0] - raw_scores[1]) > DISAGREEMENT_THRESHOLD:
            final_cn = min(raw_scores)
        else:
            final_cn = float(np.mean(raw_scores))

        # Promedio de invariant scores entre muestras
        if all_invariant_scores:
            max_len = max(len(s) for s in all_invariant_scores)
            averaged_inv = [
                round(float(np.mean([s[i] for s in all_invariant_scores if i < len(s)])), 4)
                for i in range(max_len)
            ]
        else:
            averaged_inv = []

        if last_data is None:
            last_data = {}
        raw_reasoning = (
            last_data.get("reasoning") or last_data.get("reason") or
            last_data.get("message") or last_data.get("feedback") or "No feedback provided."
        )

        verdict = JudgeResult(
            consistency_score=round(final_cn, 4),
            phantom_files=bool(last_data.get("phantom_files", False)),
            missing_acceptance=bool(last_data.get("missing_acceptance", False)),
            violates_constraints=bool(last_data.get("violates_constraints", False)),
            reasoning=str(raw_reasoning),
            invariant_scores=averaged_inv,
            sample_scores=[round(s, 4) for s in raw_scores],
        )

        return TaskResult(
            worker_id=self.role, task_id=task.id, status=TaskStatus.COMPLETED,
            notes=json.dumps(verdict.model_dump()),
            metadata=verdict.model_dump()
        )
```

---

## 4. Fisher Score Guard (`guards/fisher_scoring.py`)

```python
import numpy as np
from typing import Dict
from ps_agent.llmops.fisher import FisherCalculator


class FisherScorer:
    """
    Fisher Score compositor.

    Formula:
        Sf = 0.50 * Cn + 0.20 * (1 - Vn) + 0.30 * Qn

    Donde:
        Cn  — LLM judge consistency score (multi-sample, del EvaluatorWorker)
        Vn  — Varianza Levenshtein de shadow runs (menor = más estable)
        Qn  — Structural quality score de code_quality.compute_qn()
              (test presence, mock usage, syntax validity, arch compliance)

    `un` (epistemic uncertainty) se acepta por compat pero ya no penaliza Sf
    directamente — se usa como feature en el contexto del LinUCB bandit.
    """

    def __init__(self, config: Dict = None):
        config = config or {}
        weights = config.get("weights", {})

        # w_c  → peso de Cn en compute_sf  (correctness_w en YAML)
        # w_vn → peso de (1-Vn) en compute_sf (volatility_w en YAML)
        # w_qn → peso de Qn; derivado para que w_c+w_vn+w_qn == 1.0
        self._w_c  = float(weights.get("c",  0.50))
        self._w_vn = float(weights.get("vn", 0.20))
        self._w_qn = max(0.0, round(1.0 - self._w_c - self._w_vn, 6))

        self.calculator = FisherCalculator(
            weights={"cn": self._w_c, "vn": self._w_vn + self._w_qn}
        )

        thresholds = config.get("thresholds", {})
        self.p10 = thresholds.get("p10", 0.10)
        self.p90 = thresholds.get("p90", 0.80)

    def normalize(self, value: float) -> float:
        """Vn = clip((V - P10) / (P90 - P10), 0, 1)"""
        if value <= self.p10:
            return 0.0
        if value >= self.p90:
            return 1.0
        return float(np.clip((value - self.p10) / (self.p90 - self.p10 + 1e-9), 0, 1))

    def compute_sf(self, un: float, vn: float, c: float, qn: float = 0.0) -> float:
        """
        Calcula el Fisher Score Sf.

        Args:
            un:  Epistemic uncertainty (para compat de firma, no penaliza).
            vn:  Varianza Levenshtein de shadow runs [0, 1].
            c:   Judge consistency score Cn [0, 1].
            qn:  Structural quality score de compute_qn() [0, 1]. Default 0.0.

        Returns:
            Sf en [0.0, 1.0].
        """
        sf = self._w_c * c + self._w_vn * (1.0 - vn) + self._w_qn * qn
        return float(np.clip(sf, 0.0, 1.0))
```

---

## 5. Geometric Guard (`guards/geometric.py`)

Mide la adherencia geométrica del código generado al espacio semántico del dominio:
cosine similarity entre el embedding del código y el centroide precalculado del dominio.

```python
import numpy as np
from pathlib import Path
import logging
import yaml
from ps_agent.guards.domain_router import DomainRouter

logger = logging.getLogger(__name__)


class GeometricGuard:
    def __init__(self, llm_client):
        if llm_client is None:
            raise ValueError("GeometricGuard requires explicit llm_client injection")
        self.llm_client = llm_client

        self.config = {}
        try:
            config_path = Path(__file__).parent.parent.parent.parent / "config" / "commercial_profile_v1.yaml"
            if config_path.exists():
                with open(config_path, "r") as f:
                    self.config = yaml.safe_load(f)
        except Exception as e:
            logger.warning(f"⚠️ GeometricGuard: Failed to load config: {e}")

        default_threshold = float(self.config.get("threshold_geom", 0.679))
        self.router = DomainRouter(default_threshold=default_threshold)
        self.centroids_cache = {}

    def _load_centroid(self, domain: str, relative_path: str) -> np.ndarray:
        """Carga y cachea el centroide del dominio (dimension-agnostic)."""
        if domain in self.centroids_cache:
            return self.centroids_cache[domain]

        full_path = Path(__file__).parent / relative_path
        if not full_path.exists():
            logger.warning(f"⚠️ Centroid not found for domain '{domain}': {relative_path}")
            return None

        centroid = np.load(full_path)
        if centroid.ndim != 1 or centroid.shape[0] == 0:
            logger.error(f"❌ Centroid {domain} has invalid shape: {centroid.shape}")
            return None

        # Normalizar para cosine similarity
        norm = np.linalg.norm(centroid)
        if norm > 0:
            centroid = centroid / norm

        self.centroids_cache[domain] = centroid
        return centroid

    async def _get_vector(self, text: str) -> tuple[np.ndarray, float]:
        """Extrae y normaliza el embedding del código generado."""
        embedding = await self.llm_client.embed(text)
        if not embedding:
            raise ValueError("LLM Client returned empty embedding")

        vec = np.array(embedding, dtype=np.float32)
        norm = float(np.linalg.norm(vec))
        vec_normed = vec / norm if norm > 0 else vec
        return vec_normed, norm

    async def check_adherence(self, generated_code: str, file_path: str = None) -> dict:
        """
        Verifica adherencia geométrica contra el centroide del dominio.

        Thresholds:
          score >= threshold        → PASS
          score >= threshold * 0.9  → WARN
          score <  threshold * 0.9  → REJECTED
        """
        if not generated_code or len(generated_code.strip()) < 10:
            return {"status": "REJECTED", "score": 0.0,
                    "reason": "Code too short or empty", "metrics": {}}

        try:
            # 1. Resolver dominio vía DomainRouter
            route_info = self.router.resolve(file_path)
            domain = route_info["domain"]
            centroid_path = route_info["centroid_path"]
            threshold = float(route_info["threshold"])

            # 2. Cargar centroide
            centroid = self._load_centroid(domain, centroid_path)
            if centroid is None:
                # Dormant mode: sin centroide → SKIPPED (fail open)
                return {"status": "SKIPPED",
                        "reason": f"No centroid loaded for domain: {domain}",
                        "score": 1.0, "metrics": {"domain": domain}}

            # 3. Extraer vector del código generado
            vec_new, norm = await self._get_vector(generated_code)

            if vec_new.shape[0] != centroid.shape[0]:
                raise ValueError(
                    f"Vector/Centroid Dimension Mismatch: {vec_new.shape} vs {centroid.shape}"
                )

            # 4. Cosine Similarity (dot product de vectores normalizados)
            score = float(np.dot(vec_new, centroid))

            # 5. Threshold logic con soft warning zone
            soft_threshold = threshold * 0.9
            if score >= threshold:
                status = "PASS"
            elif score >= soft_threshold:
                status = "WARN"
            else:
                status = "REJECTED"

            return {
                "status": status,
                "score": score,
                "reason": f"Geometric Score {score:.4f} vs Threshold {threshold} (Domain: {domain})",
                "metrics": {
                    "embedding_norm": norm,
                    "geometric_score": score,
                    "domain": domain,
                    "domain_threshold": threshold
                }
            }

        except Exception as e:
            logger.error(f"❌ Geometric Check Critical Failure: {e}")
            return {"status": "SKIPPED", "reason": f"Guard Error: {e}",
                    "score": 0.0, "metrics": {}}
```

---

## 6. Epistemic Uncertainty Guard (`guards/uncertainty.py`)

Mide si el modelo "sabe lo que hace" antes de ejecutar.
Genera N explicaciones técnicas y mide su divergencia semántica en el espacio de embeddings.

```python
import numpy as np
from typing import List


class UncertaintyScorer:
    def __init__(self, client):
        if client is None:
            raise ValueError("UncertaintyScorer requires explicit client injection")
        self.client = client

        # Parámetros de calibración empírica
        self.mu = 0.18      # Media esperada de avg_dist (tareas bien definidas)
        self.sigma = 0.06   # Desviación estándar empírica
        self.eps = 1e-8     # Estabilidad numérica

    async def compute_un(self, texts: List[str]) -> float:
        """
        Calcula la Incertidumbre Epistémica (Un).

        Formula:
            d_i   = 1 - cos(ê_i, centroid)         ← distancia coseno al centroide
            d̄     = mean(d_i)                       ← distancia media
            z     = (d̄ - μ) / σ                    ← Z-score (μ=0.18, σ=0.06)
            Un    = σ(z) = 1 / (1 + e^(-z))        ← Sigmoid → [0, 1]

        Returns:
            float: 0.0 (certeza) → 1.0 (alta incertidumbre)
            Un > 0.90 con plan ambiguo → pipeline solicita clarificación
        """
        if not texts or len(texts) < 2:
            return 0.0

        # 1. Embeddings normalizados L2
        embeddings = []
        for text in texts:
            vec = await self.client.embed(text)
            if vec is not None:
                vec = np.array(vec, dtype=np.float32)
                norm = np.linalg.norm(vec)
                if norm > self.eps:
                    vec = vec / norm
                embeddings.append(vec)

        if len(embeddings) < 2:
            return 1.0  # fallback conservador

        matrix = np.vstack(embeddings)

        # 2. Centroide normalizado
        centroid = np.mean(matrix, axis=0)
        norm_centroid = np.linalg.norm(centroid)
        if norm_centroid > self.eps:
            centroid = centroid / norm_centroid

        # 3. Distancias coseno al centroide
        distances = []
        for vec in matrix:
            cos_sim = np.clip(np.dot(vec, centroid), -1.0, 1.0)
            distances.append(1.0 - cos_sim)

        avg_dist = float(np.mean(distances))

        # 4. Normalización estadística: Z-score + Sigmoid
        z = (avg_dist - self.mu) / (self.sigma + self.eps)
        un_score = 1.0 / (1.0 + np.exp(-z))

        return float(np.clip(un_score, 0.0, 1.0))
```

---

## Resumen de guards: qué mide cada uno

| Guard | Señal | Cuándo se ejecuta | Umbral |
|---|---|---|---|
| **UncertaintyScorer** | Divergencia semántica de N planes del LLM | Pre-ejecución | `Un > 0.90` → clarificación |
| **FisherScorer** | `Sf = 0.50·Cn + 0.20·(1-Vn) + 0.30·Qn` | Post-ejecución | `Sf ≥ 0.70` → aplicar parche |
| **GeometricGuard** | Cosine similarity código vs centroide del dominio | Post-generación | backend: 0.679, frontend: 0.650 |

Los tres miden cosas distintas del espacio de embeddings:
- **Un**: divergencia entre explicaciones *del task* (¿entiende lo que tiene que hacer?)
- **Sf/Cn**: consistencia del *código generado* contra los invariantes (¿lo hizo bien?)
- **Geo**: distancia del *código generado* al centroide del dominio (¿suena a nuestro codebase?)

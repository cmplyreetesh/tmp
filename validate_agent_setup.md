# FINAL PRE-FLIGHT CHECK — Ensure everything runs smoothly
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# ════════════════════════════════════════════════════════════════════════════

@workspace

Read ALL of these files before responding. Do not write any code until you 
have read every file listed. Do NOT run any code or terminal commands.

Files to read:
- #file:agent_setup.py
- #file:ModelConfig.py  
- #file:app.py
- #file:backend/agents/intent_agent.py
- #file:backend/agents/selector_agent.py
- #file:backend/routers/query_router.py
- #file:backend/processors/sherlock_processor.py
- #file:backend/processors/response_builder.py
- #file:backend/gates/dq_gate.py
- #file:backend/gates/recon_gate.py

---

## ACTUAL PROJECT STRUCTURE (confirmed from file explorer)

The project root is: H:\work\AI-ML\Final_agent\backend\sandbox_backend\

```
sandbox_backend\                     ← PROJECT ROOT (all files run from here)
├── agent_setup.py                   ← Agent + get_domestic_metadata()
├── ModelConfig.py                   ← smart_sdk Bedrock auth, login(), model
├── app.py                           ← Entry point, LocalRunner, runner.run()
├── main.py                          ← IGNORE — standalone FastAPI, not used
├── requirements.txt
├── synthetic_data.py
└── backend\                         ← Sherlock computation modules
    ├── agents\
    │   ├── __init__.py
    │   ├── intent_agent.py
    │   ├── selector_agent.py
    │   ├── financial_agent.py
    │   └── overdraft_agent.py
    ├── data\
    │   ├── internal_analytics_data.csv
    │   └── synthetic_financial_data_1.xlsx
    ├── gates\
    │   ├── __init__.py
    │   ├── dq_gate.py
    │   └── recon_gate.py
    ├── models\
    │   ├── __init__.py
    │   └── response_models.py
    ├── processors\
    │   ├── __init__.py
    │   ├── response_builder.py
    │   └── sherlock_processor.py
    ├── routers\
    │   ├── __init__.py
    │   └── query_router.py
    └── utils\
        ├── __init__.py
        └── llm_sandbox.py
```

---

## CHECK 1 — sys.path in agent_setup.py

The `_BACKEND` path must resolve to the `backend\` subfolder.
Since agent_setup.py is in `sandbox_backend\`, the correct path is:

```python
_BACKEND = os.path.join(os.path.dirname(os.path.abspath(__file__)), "backend")
```

Verify:
- Does `os.path.dirname(os.path.abspath(__file__))` resolve to `sandbox_backend\`?
- Does joining with `"backend"` give `sandbox_backend\backend\`?
- Does `sandbox_backend\backend\` actually exist in the workspace? (YES — confirmed)
- Is there a guard: `if _BACKEND not in sys.path`?

If the folder name used is different (e.g. "sandbox_backend" instead of "backend"),
flag it and show the correction.

---

## CHECK 2 — Data file path in sherlock_processor.py

Read #file:backend/processors/sherlock_processor.py and find DATA_PATH.

The data files in the workspace are:
- `backend\data\internal_analytics_data.csv`
- `backend\data\synthetic_financial_data_1.xlsx`

Note: the xlsx file is named `synthetic_financial_data_1.xlsx` (has `_1` suffix).

Verify:
- Does DATA_PATH match the ACTUAL filename exactly (including `_1` if present)?
- Does the fallback chain try CSV if xlsx not found?
- If DATA_PATH is wrong, show the corrected line only.

---

## CHECK 3 — app.py flow is complete and unchanged

Read #file:app.py and confirm this exact flow exists:

```
execute_query(query, user_id, session_id)
  └── agent = lbp_cash_investigation_agent(model)
  └── runner = LocalRunner(app_name=..., user_id=..., session_id=...)
  └── response = runner.run(agent, query)
  └── return response
```

Verify:
- `from agent_setup import lbp_cash_investigation_agent` — present?
- `from ModelConfig import model` — present?
- `LocalRunner` imported from smart_sdk — present?
- `runner.run(agent, query)` — present?
- NO imports from `backend\` folder in app.py — confirmed?
- NO HTTP calls in app.py — confirmed?

---

## CHECK 4 — ModelConfig.py has Bedrock login

Read #file:ModelConfig.py and confirm:
- `login()` is called to get `gateway_token`
- `Model(provider=ModelProvider.BEDROCK, ...)` is used
- Module-level `model = get_model()` is exported
- NO reference to ANTHROPIC_API_KEY (that's the old standalone version)

---

## CHECK 5 — utils/llm_sandbox.py (new file — what is it?)

Read `backend/utils/llm_sandbox.py`.

This file was not in the original Sherlock backend design.
Determine:
- What does it do?
- Is it imported anywhere in the pipeline (intent_agent, sherlock_processor, etc.)?
- If it imports `agents.model_config` or `anthropic` — flag it as a problem.
- If it is not imported anywhere — it is dead code, note it but do not remove.

---

## CHECK 6 — No circular imports

Trace the import chain from agent_setup.py:

```
agent_setup.py
  ├── from ModelConfig import model
  ├── from agents.intent_agent import detect_intent
  │     └── does intent_agent import anything from agent_setup? (should be NO)
  ├── from processors.sherlock_processor import process, _load, _cast, _derive
  │     └── does sherlock_processor import ModelConfig? (should be NO)
  └── from processors.response_builder import build_response
        └── does response_builder import ModelConfig? (should be NO)
```

Flag any circular import (A imports B, B imports A).

---

## CHECK 7 — requirements.txt has all needed packages

Read #file:requirements.txt and verify these are ALL present:
- `fastapi` or `fastapi==...`
- `pandas`
- `numpy`  
- `openpyxl`        ← required for pd.read_excel()
- `python-dotenv`
- `pydantic`
- smart_sdk package (whatever it is named in your requirements)

Flag any missing package that would cause an ImportError at startup.
Do NOT flag packages used only by main.py (uvicorn, etc.) — those are fine to keep.

---

## FINAL OUTPUT

Give me a table like this:

| Check | Status | Issue (if any) | Fix needed |
|-------|--------|----------------|------------|
| 1. sys.path | ✅/❌ | ... | ... |
| 2. Data file path | ✅/❌ | ... | ... |
| 3. app.py flow | ✅/❌ | ... | ... |
| 4. Bedrock login | ✅/❌ | ... | ... |
| 5. llm_sandbox.py | ✅/❌ | ... | ... |
| 6. No circular imports | ✅/❌ | ... | ... |
| 7. requirements.txt | ✅/❌ | ... | ... |

Then:
- If ALL green: output exactly "✅ READY TO RUN — execute app.py"
- If any red: show ONLY the specific lines that need changing, nothing else.
  Do not rewrite entire files. One fix at a time.

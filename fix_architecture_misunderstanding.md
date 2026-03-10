# CORRECTION PROMPT — Fix architecture misunderstanding
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# Files needed open: agent_setup.py, ModelConfig.py, app.py,
#                    backend/processors/sherlock_processor.py
# ════════════════════════════════════════════════════════════════════════════

@workspace

You have made a critical architecture error in your previous response.
Read #file:agent_setup.py, #file:ModelConfig.py, and #file:app.py carefully again.

---

## THE WRONG ASSUMPTION YOU MADE

You showed this architecture:
```
Main Agent (agent_with_audit)     HTTP request     Backend API (sandbox_backend)
  - Uses ModelConfig.py        ───────────────►   - NO ModelConfig.py
  - Calls get_domestic_metadata()                  - Pure computation
```

THIS IS COMPLETELY WRONG. There is NO HTTP. There is NO separate server.
There is NO FastAPI running. The backend/main.py server is NOT used at all.

---

## THE CORRECT ARCHITECTURE

The Sherlock backend Python files are imported DIRECTLY into agent_setup.py
as regular Python modules — the same way you import any Python file.

```
agent_with_audit/
│
├── ModelConfig.py              ← smart_sdk Bedrock auth, login(), model instance
├── app.py                      ← LocalRunner, runner.run(agent, query)
├── agent_setup.py              ← Agent definition, get_domestic_metadata()
│
└── backend/                    ← Sherlock backend — imported DIRECTLY, not HTTP
    ├── agents/intent_agent.py         ← imported as: from agents.intent_agent import detect_intent
    ├── agents/selector_agent.py       ← imported as: from agents.selector_agent import enrich_context
    ├── routers/query_router.py        ← imported as: from routers.query_router import build_query
    ├── processors/sherlock_processor.py  ← imported as: from processors.sherlock_processor import process
    ├── processors/response_builder.py   ← imported as: from processors.response_builder import build_response
    ├── gates/dq_gate.py               ← imported as: from gates.dq_gate import run_dq_gate
    └── gates/recon_gate.py            ← imported as: from gates.recon_gate import run_recon_gate
```

The call flow is:
```
app.py
  └── runner.run(agent, query)                    ← smart_sdk LocalRunner, unchanged
        └── LLM (Bedrock via ModelConfig.py)      ← ONLY LLM call, uses Bedrock login()
              └── calls get_domestic_metadata()   ← tool in agent_setup.py
                    └── DIRECT Python import calls:
                          detect_intent()         ← pure regex, no LLM
                          build_query()           ← pure Python
                          enrich_context()        ← pure Python
                          process()               ← pure pandas/numpy
                          run_dq_gate()           ← pure pandas
                          run_recon_gate()        ← pure pandas
                          build_response()        ← pure Python dict assembly
                    └── returns tool_data dict to LLM
              └── LLM formats HTML using tool_data ← narration only
```

---

## THE ROLE OF ModelConfig.py

`ModelConfig.py` is used EXACTLY as before — nothing changes:
- `login()` is called at startup to authenticate with Bedrock via smart_sdk
- `model = get_model()` returns the authenticated smart_sdk Model instance
- `agent_setup.py` does `from ModelConfig import model` — same as always
- The LLM (Bedrock) is called by smart_sdk's `runner.run(agent, query)` — same as always

The Sherlock backend files (sherlock_processor.py, intent_agent.py etc.) do NOT use
ModelConfig.py at all — they are pure Python computation, zero LLM calls.

---

## TASK

Fix `agent_setup.py` to match the correct architecture:

1. The sys.path.insert must add the `backend/` folder so Python can find the backend modules:
```python
import sys, os
_BACKEND = os.path.join(os.path.dirname(os.path.abspath(__file__)), "backend")
if _BACKEND not in sys.path:
    sys.path.insert(0, _BACKEND)
```

2. Import the backend modules directly (NO http, NO requests, NO FastAPI client):
```python
from agents.intent_agent     import detect_intent
from agents.selector_agent   import enrich_context
from routers.query_router    import build_query
from processors.sherlock_processor import process, _load, _cast, _derive
from processors.response_builder   import build_response
from gates.dq_gate    import run_dq_gate
from gates.recon_gate import run_recon_gate
```

3. `get_domestic_metadata(query)` calls these functions directly — no HTTP, no await, no requests:
```python
def get_domestic_metadata(query: str) -> dict:
    intent_result  = detect_intent(query)
    data_query     = build_query(intent_result)
    context        = enrich_context(data_query, intent_result)
    process_result = process(data_query, context)
    full_df        = _derive(_cast(_load()))
    dq_header      = run_dq_gate(full_df)
    recon_result   = run_recon_gate(full_df)
    tool_data      = build_response(intent_result, process_result, dq_header, recon_result, data_query)
    return tool_data
```

4. `from ModelConfig import model` stays — unchanged, still used for Bedrock auth
5. `lbp_cash_investigation_agent(model)` stays — unchanged
6. `Agent(name=..., model=model, tools=[get_domestic_metadata])` stays — unchanged

---

## VALIDATION

After fixing, confirm:
✅ No `import requests` or `httpx` or any HTTP client in agent_setup.py
✅ No `http://localhost` or any URL in agent_setup.py
✅ `from ModelConfig import model` still present
✅ backend modules imported via sys.path, NOT via HTTP
✅ `get_domestic_metadata()` is a plain synchronous Python function — no async, no await
✅ `ModelConfig.py` and `app.py` are UNCHANGED
✅ Bedrock login() still happens via ModelConfig.py at startup

Do NOT suggest running any code or terminal commands. Read files only.
Show me the corrected agent_setup.py only.

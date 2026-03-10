# VALIDATION PROMPT — Verify agent_setup.py is correct before testing
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# Files needed open: agent_setup.py, ModelConfig.py, app.py,
#                    backend/agents/intent_agent.py,
#                    backend/routers/query_router.py,
#                    backend/processors/sherlock_processor.py,
#                    backend/processors/response_builder.py,
#                    backend/gates/dq_gate.py,
#                    backend/gates/recon_gate.py
# ════════════════════════════════════════════════════════════════════════════

@workspace

Read #file:agent_setup.py completely. Then read each backend file as needed to validate.
Do NOT suggest running any code or terminal commands. Static analysis only.

Validate all 6 sections below and give a PASS/FAIL for each check.
At the end, give a final verdict: READY TO TEST or NEEDS FIX.

---

## SECTION 1 — sys.path (Will Python find the backend modules?)

Read the sys.path.insert line in agent_setup.py.

CHECK 1.1: Does it use `os.path.abspath(__file__)` to build the path? (not a hardcoded string)
CHECK 1.2: Does it join with `"backend"` (or the actual folder name shown in the Expected Structure)?
           The folder is called `sandbox_backend` based on Copilot's output — verify which name is used
           and whether that folder actually exists in the workspace.
CHECK 1.3: Is there a guard `if _BACKEND not in sys.path` to avoid duplicate inserts?

---

## SECTION 2 — Imports (Are all 7 backend modules imported correctly?)

CHECK 2.1: `from agents.intent_agent import detect_intent` — present?
CHECK 2.2: `from agents.selector_agent import enrich_context` — present?
CHECK 2.3: `from routers.query_router import build_query` — present?
CHECK 2.4: `from processors.sherlock_processor import process, _load, _cast, _derive` — present?
CHECK 2.5: `from processors.response_builder import build_response` — present?
CHECK 2.6: `from gates.dq_gate import run_dq_gate` — present?
CHECK 2.7: `from gates.recon_gate import run_recon_gate` — present?
CHECK 2.8: `from ModelConfig import model` — still present and unchanged?

RED FLAGS — fail immediately if any of these exist:
❌ `import requests` or `import httpx` — means HTTP is being used
❌ `http://localhost` or any URL string — means HTTP is being used
❌ `from agents.model_config import` — old Anthropic client, must be gone
❌ `import anthropic` — wrong LLM client, must be gone
❌ `async def get_domestic_metadata` — must be synchronous, not async

---

## SECTION 3 — Pipeline order in get_domestic_metadata()

Read #file:backend/processors/sherlock_processor.py to confirm function names exist.
Read #file:backend/processors/response_builder.py to confirm build_response signature.

CHECK 3.1: Does get_domestic_metadata() call exactly these steps in this order?
  Step 1: intent_result  = detect_intent(query)
  Step 2: data_query     = build_query(intent_result, ...)
  Step 3: context        = enrich_context(data_query, intent_result)
  Step 4: process_result = process(data_query, context)
  Step 5: full_df        = _derive(_cast(_load()))
  Step 6: dq_header      = run_dq_gate(full_df)
  Step 7: recon_result   = run_recon_gate(full_df)
  Step 8: tool_data      = build_response(intent_result, process_result, dq_header, recon_result, data_query)
  Step 9: return tool_data

CHECK 3.2: Read the ACTUAL signature of build_response() in response_builder.py.
           Do the argument names and ORDER in the call match the function signature exactly?
           List both the call and the signature side by side.

CHECK 3.3: Does get_domestic_metadata() contain any arithmetic? (+, -, *, /, sum, count, mean)
           If yes — FAIL. All math must be in sherlock_processor.py only.

---

## SECTION 4 — Agent definition unchanged

CHECK 4.1: Is `lbp_cash_investigation_agent(model)` function signature unchanged?
CHECK 4.2: Does it return `Agent(name=..., model=model, tools=[get_domestic_metadata], system_message=...)`?
CHECK 4.3: Is the `tools` list exactly `[get_domestic_metadata]`? (not a list of strings, not HTTP endpoints)
CHECK 4.4: Does system_message call `get_sherlock_system_prompt()`?
CHECK 4.5: Does `get_sherlock_system_prompt()` contain the phrase "PRESENTATION LAYER ONLY"?
CHECK 4.6: Does `get_sherlock_system_prompt()` contain at least one ❌ prohibition on arithmetic?
CHECK 4.7: Does `get_sherlock_system_prompt()` preserve the original VISUALIZATION GUIDELINES
           section (the ```json chart format block)?

---

## SECTION 5 — ModelConfig.py and app.py are untouched

CHECK 5.1: Read #file:ModelConfig.py — does it still have `login()` and `get_model()` using smart_sdk?
CHECK 5.2: Read #file:app.py — does it still have `LocalRunner` and `runner.run(agent, query)`?
CHECK 5.3: Neither file imports anything from the backend/ folder?

---

## SECTION 6 — Trace a sample query end-to-end (dry run, no execution)

Trace what happens when this query is submitted:
  "show top 5 overdraft accounts"

Using only static code reading (no execution), answer:

TRACE 6.1: What does detect_intent("show top 5 overdraft accounts") return?
           Read intent_agent.py regex patterns — which pattern matches? What intent name?

TRACE 6.2: What does build_query() return for that intent?
           Read query_router.py — what are sort_by, od_only, top_n values?

TRACE 6.3: Does enrich_context() route to FinancialAgent or OverdraftAgent for this intent?
           Read selector_agent.py — check OD_INTENTS set.

TRACE 6.4: What columns does process() use for filtering?
           Read sherlock_processor.py — does it filter od_only=True and sort by OD_AMOUNT desc?

TRACE 6.5: What does build_response() return?
           Read response_builder.py — does viz.primary use OD_AMOUNT (not EFF_BALANCE_LCY)?

---

## FINAL VERDICT

After all 6 sections, give one of:

✅ READY TO TEST — all checks pass, architecture is correct
⚠️ NEEDS MINOR FIX — list specific lines to change (do not rewrite whole file)
❌ NEEDS REWORK — critical issue found, describe it clearly

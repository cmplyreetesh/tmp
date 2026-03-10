# FINAL WORKING VALIDATION — Full end-to-end review
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# Have ALL files open in editor before running
# ════════════════════════════════════════════════════════════════════════════

@workspace

Read every file listed below completely before writing any response.
Do NOT run code. Do NOT make any changes. Static analysis only.

Files to read:
- #file:main.py
- #file:ModelConfig.py
- #file:app.py
- #file:agent_setup.py
- #file:backend/agents/intent_agent.py
- #file:backend/agents/selector_agent.py
- #file:backend/routers/query_router.py
- #file:backend/processors/sherlock_processor.py
- #file:backend/processors/response_builder.py
- #file:backend/gates/dq_gate.py
- #file:backend/gates/recon_gate.py
- #file:backend/models/response_models.py

---

## TRACE 1 — Startup (what happens when python main.py runs)

Read main.py top-level code and confirm in order:
1. Does `from ModelConfig import model` execute at import time?
2. Does ModelConfig.py call `login()` and `get_model()` at module level?
3. Does `from agent_setup import lbp_cash_investigation_agent` succeed?
4. Does agent_setup.py sys.path correctly point to the `backend\` folder?
5. Do all 7 backend imports in agent_setup.py resolve without error?
   - from agents.intent_agent import detect_intent
   - from agents.selector_agent import enrich_context
   - from routers.query_router import build_query
   - from processors.sherlock_processor import process, _load, _cast, _derive
   - from processors.response_builder import build_response
   - from gates.dq_gate import run_dq_gate
   - from gates.recon_gate import run_recon_gate
6. Does FastAPI app start with correct CORS and routes?

Flag any import that will fail. Show the exact error and fix.

---

## TRACE 2 — React submits query (POST /api/query/submit)

Trace what happens when React POSTs:
```json
{
  "prompt": "show top 5 overdraft accounts",
  "user_id": "user1",
  "page": 1,
  "page_size": 50
}
```

Verify:
- Does the SubmitRequest Pydantic model accept this shape?
- Does submit_query() create a job_id and return 202 with {job_id, status, progress}?
- Does asyncio.create_task(_run_pipeline(...)) fire correctly?

---

## TRACE 3 — _run_pipeline execution (all 7 steps)

Trace each step for query "show top 5 overdraft accounts":

STEP 1 — detect_intent("show top 5 overdraft accounts")
- Read intent_agent.py regex patterns
- Which pattern matches? What intent name is returned?
- What does the full return dict look like?

STEP 2 — build_query(intent_result, page=1, page_size=50)
- Read query_router.py for "top_overdraft" intent
- What are: sort_by, sort_dir, od_only, top_n, viz_type values?

STEP 3 — enrich_context(query, intent_result)
- Read selector_agent.py — is "top_overdraft" in OD_INTENTS?
- Which agent handles it: FinancialAgent or OverdraftAgent?
- What display_columns does it set?

STEP 4 — process(query, context)
- Read sherlock_processor.py process() function
- Does it filter od_only=True correctly?
- Does it sort by OD_AMOUNT descending?
- Does it return top 5 rows?
- Are all 8 derived columns present in each row?
  (IS_OVERDRAFT, OD_AMOUNT, EFF_BALANCE_DISPLAY, OD_TENURE_DAYS,
   OD_STATUS, OD_STATUS_COLOR, OD_STATUS_ICON, OD_TENURE_LABEL)

STEP 5 — run_dq_gate(full_df) and run_recon_gate(full_df)
- Does dq_gate return all 9 required keys?
  (score, max_score, label, passed, records, critical, warnings, data_as_of, display)
- Does recon_gate return passed boolean and checks list?

STEP 6 — build_response(intent_result, process_result, dq_header, recon_result, query)
- Read response_builder.py — does the argument ORDER match this call exactly?
- Does it return ALL required top-level keys?
  (status, dq_header, executive_summary, key_insights, gate_checks,
   details, aggregates, viz, metadata)
- Does viz.primary use OD_AMOUNT (not EFF_BALANCE_LCY) for top_overdraft?
- Does viz.aging always exist (even if empty)?
- Does details.columns include the special "od_status" field for OD_TENURE_LABEL?

STEP 7 — LLM narration via runner.run(agent, narrative_prompt)
- Does it create agent via lbp_cash_investigation_agent(model)?
- Does it create runner via create_stateless_runner()?
- Does the narrative_prompt pass the precomputed tool_data to the LLM?
- Does it parse the JSON response to extract executive_summary and key_insights?
- If JSON parsing fails — does it gracefully fall back (no crash)?
- After step 7, does result["executive_summary"] contain a string (not HTML)?
- After step 7, does result["key_insights"] contain a list of strings?

---

## TRACE 4 — React polls for result (GET /api/query/{job_id}/status)

Verify:
- Does the status endpoint return {job_id, status, progress, result, error}?
- When status="complete", does result contain the full JSON?
- Does the JSON shape match what React renderResult() expects?

Cross-check these specific fields React reads against what the API returns:

| React reads | API must return | Match? |
|-------------|-----------------|--------|
| r.dq_header.score | int | ? |
| r.dq_header.display | string | ? |
| r.dq_header.critical | int | ? |
| r.dq_header.warnings | int | ? |
| r.executive_summary | plain text string | ? |
| r.key_insights | array of strings | ? |
| r.details.columns | array of {field,headerName,sortable,filter,width} | ? |
| r.details.rows | array of account objects | ? |
| r.details.showing_label | string | ? |
| r.aggregates.total_account_count | int | ? |
| r.aggregates.overdraft_account_count | int | ? |
| r.aggregates.highest_od_client | string | ? |
| r.aggregates.highest_od_amount | float | ? |
| r.aggregates.total_od_amount | float | ? |
| r.viz.primary | Highcharts config object | ? |
| r.viz.aging | Highcharts config object | ? |
| r.metadata.intent | string | ? |
| r.metadata.intent_source | string | ? |
| r.metadata.generated_at | string | ? |

Flag any mismatch between what React expects and what the API actually returns.

---

## TRACE 5 — Error handling (what happens when things go wrong)

Verify these failure cases are handled:

5.1 Data file missing (xlsx and csv both absent)
- Does sherlock_processor fall back to synthetic_data.py?
- Does it log a warning (not crash)?

5.2 Intent not recognized by regex
- Does intent_agent return {"intent": "overview", "source": "fallback"}?
- Does the pipeline continue correctly with "overview" intent?

5.3 LLM narration fails (step 7 raises exception)
- Does _run_pipeline catch it gracefully?
- Does result still have executive_summary="" and key_insights=[]?
- Is the job marked "complete" (not "failed") since data is still valid?

5.4 DQ gate fails (score < 80)
- Does dq_header.passed = False?
- Does gate_checks.dq.passed = False in the response?
- Does React frontend receive the full JSON (React handles display, not backend)?

---

## FINAL VERDICT TABLE

| Area | Status | Issue | Fix |
|------|--------|-------|-----|
| Startup imports | ✅/❌ | | |
| POST /submit | ✅/❌ | | |
| Step 1: detect_intent | ✅/❌ | | |
| Step 2: build_query | ✅/❌ | | |
| Step 3: enrich_context | ✅/❌ | | |
| Step 4: process() | ✅/❌ | | |
| Step 5: gates | ✅/❌ | | |
| Step 6: build_response | ✅/❌ | | |
| Step 7: LLM narration | ✅/❌ | | |
| GET /status response | ✅/❌ | | |
| React JSON shape | ✅/❌ | | |
| Error handling | ✅/❌ | | |

If ALL green → output exactly:
✅ READY TO RUN — start with: python main.py

If any red → show ONLY the specific line(s) to fix.
Do not rewrite entire files. One targeted fix per issue.

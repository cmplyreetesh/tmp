# COPILOT PROMPT — Wire ModelConfig + agent into main.py, return JSON to React
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# Files needed open: main.py, ModelConfig.py, app.py, agent_setup.py,
#                    backend/processors/response_builder.py
# ════════════════════════════════════════════════════════════════════════════

@workspace

Read ALL of these files before writing any code:
- #file:main.py
- #file:ModelConfig.py
- #file:app.py
- #file:agent_setup.py
- #file:backend/processors/response_builder.py

Do NOT run any code. Do NOT restructure files. Do NOT move files.

---

## CONTEXT — What each file does

main.py:
- FastAPI server with async job queue (_run_pipeline)
- Already runs the Sherlock pipeline: detect_intent → build_query → process → build_response
- build_response already returns a complete JSON dict with these keys:
    dq_header, details, aggregates, viz, metadata, gate_checks
- BUT: executive_summary and key_insights are currently empty placeholders
- MISSING: LLM agent call using ModelConfig.py model

ModelConfig.py:
- Handles Bedrock auth via login()
- Exports: model = get_model()  ← authenticated smart_sdk Model instance

app.py:
- Shows the pattern for calling the agent:
    agent  = lbp_cash_investigation_agent(model)
    runner = StatelessRunner(...)
    result_text = runner.run(agent, query)

agent_setup.py:
- get_domestic_metadata(query) — the tool the LLM calls
- lbp_cash_investigation_agent(model) — creates the Agent with that tool
- get_sherlock_system_prompt() — narration-only system prompt

---

## THE PROBLEM

Currently _run_pipeline in main.py:
1. Runs Sherlock pipeline → gets tool_data JSON ✅
2. Stores result = build_response(...) ✅
3. Returns JSON to React ✅

BUT it never calls the LLM agent, so:
- executive_summary = "" (empty)
- key_insights = [] (empty)
- The LLM never narrates the data

---

## WHAT TO CHANGE — main.py only

### CHANGE 1: Import model from ModelConfig at top of main.py

Add after existing imports:
```python
from ModelConfig import model
from agent_setup import lbp_cash_investigation_agent
```

### CHANGE 2: Add LLM narration step inside _run_pipeline

After step 6 (build_response), add a step 7 that:
1. Creates the agent using app.py pattern
2. Calls runner.run(agent, query) — the LLM calls get_domestic_metadata internally
   which returns the same tool_data JSON
3. Parses the LLM response to extract ONLY executive_summary and key_insights
4. Injects them back into the result JSON

The LLM response will contain narrative text. Extract from it:
- A 2-3 sentence executive_summary paragraph
- 3-5 bullet point key_insights as a list of strings

### CHANGE 3: Return pure JSON — NO HTML strings

The React frontend reads these specific JSON fields:

```javascript
// Frontend reads:
r.dq_header          // { score, label, passed, display, critical, warnings }
r.executive_summary  // string — plain text, NO html tags
r.key_insights       // string[] — plain text bullet points, NO html tags  
r.details.columns    // [{ field, headerName, sortable, filter, width }]
r.details.rows       // [{ ...account data with all derived columns }]
r.aggregates         // { total_account_count, overdraft_account_count, ... }
r.viz.primary        // Highcharts config object
r.viz.aging          // Highcharts config object
r.metadata           // { intent, intent_source, generated_at }
r.gate_checks        // { dq: {passed}, reconciliation: {passed} }
```

IMPORTANT: executive_summary and key_insights must be PLAIN TEXT strings.
No <div>, no <table>, no <ul>, no HTML whatsoever.
The React frontend renders these as text — HTML tags would show as raw text.

### HOW TO EXTRACT narrative from LLM response

After runner.run(agent, query) returns result_text (a string), 
inject it as executive_summary into the result dict:

```python
# Step 7: LLM narration
from stateless_runner import create_stateless_runner

agent  = lbp_cash_investigation_agent(model)
runner = create_stateless_runner(
    app_name=f"Sherlock_{job_id[:8]}",
    user_id=request.user_id,
    session_id=job_id
)

# Pass the precomputed tool_data to the agent via the query
narrative_prompt = f"""
User query: {request.prompt}

The backend has already computed all metrics. Here is the tool_data:
{json.dumps(result, default=str)[:3000]}

Return a JSON object with exactly these two keys:
{{
  "executive_summary": "2-3 sentence plain text summary of findings",
  "key_insights": ["insight 1", "insight 2", "insight 3"]
}}

Return ONLY the JSON object. No markdown. No explanation.
"""

narrative_response = runner.run(agent, narrative_prompt)
narrative_text = str(narrative_response) if narrative_response else ""

# Parse the JSON from LLM response
import re, json as json_lib
try:
    # Strip markdown fences if present
    clean = re.sub(r"```(?:json)?", "", narrative_text).strip().rstrip("```").strip()
    narrative_json = json_lib.loads(clean)
    result["executive_summary"] = narrative_json.get("executive_summary", "")
    result["key_insights"]      = narrative_json.get("key_insights", [])
except Exception:
    # If parsing fails, use raw text as summary
    result["executive_summary"] = narrative_text[:500] if narrative_text else ""
    result["key_insights"]      = []

JOBS[job_id]["progress"] = 95
```

---

## VALIDATION

After changes, confirm in main.py:
✅ `from ModelConfig import model` — top-level import
✅ `from agent_setup import lbp_cash_investigation_agent` — top-level import  
✅ _run_pipeline step 7 calls runner.run() after build_response
✅ result["executive_summary"] is a plain text string (no HTML)
✅ result["key_insights"] is a list of plain text strings (no HTML)
✅ All other result keys (dq_header, details, aggregates, viz, metadata) unchanged
✅ FastAPI routes unchanged
✅ Job queue and progress tracking unchanged
✅ No changes to any other file

Show me only the updated _run_pipeline function and the new imports.
Do not show the entire main.py.

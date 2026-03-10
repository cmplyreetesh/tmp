# FIX PROMPT — coroutine 'LocalRunner.run' was never awaited
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# ════════════════════════════════════════════════════════════════════════════

@workspace

Read #file:main.py — find Step 7 (LLM narration) inside _run_pipeline.

## THE WARNING
```
RuntimeWarning: coroutine 'LocalRunner.run' was never awaited
return self._loop.run_until_complete(task)
```

## THE CAUSE
runner.run() returns a coroutine (it is async) but Step 7 calls it like:
```python
narrative_response = runner.run(agent, narrative_prompt)  # ← missing await
```

## THE FIX — 2 changes only

### Fix A: await the runner.run() call
```python
narrative_response = await runner.run(agent, narrative_prompt)
```

### Fix B: handle the response correctly
smart_sdk runner.run() returns a response object, not a plain string.
Extract the text like your original app.py does:

```python
import inspect
if inspect.iscoroutine(narrative_response):
    narrative_response = await narrative_response
narrative_text = str(narrative_response) if narrative_response else ""
```

## RULES
- Change ONLY the runner.run() line and response extraction in Step 7
- _run_pipeline is already async def — so await is valid here
- Do NOT change any other part of main.py
- Do NOT change any other file

Show only the corrected Step 7 block.

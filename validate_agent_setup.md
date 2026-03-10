# FINAL CHECK — 2 items only (everything else already confirmed green)
# ════════════════════════════════════════════════════════════════════════════
# Paste into VS Code Copilot Chat (Ctrl+Alt+I)
# ════════════════════════════════════════════════════════════════════════════

@workspace

Read #file:app.py and #file:agent_setup.py only.
Do NOT run any code or terminal commands. Static analysis only.

---

## CHECK 1 — app.py imports match what agent_setup.py exports

Read app.py and find every import it takes from agent_setup.py.
Then read agent_setup.py and confirm each imported name actually exists.

Specifically verify:
- Whatever function app.py imports from agent_setup — does it exist by that exact name?
- app.py was copied from a project called `agent_with_audit` where the agent function
  was named `lbp_cash_investigation_agent`. 
- In the new agent_setup.py, is the function still named `lbp_cash_investigation_agent`?
- If the name changed — show the one line fix in app.py.

---

## CHECK 2 — app.py imports match what agent_setup.py exports for audit/session modules

app.py likely imports audit_logger, ConversationMemory, StatelessRunner.
These come from your existing project modules (audit_service, etc.), NOT from the
Sherlock backend folder.

Verify:
- Do those imports still resolve correctly from the sandbox_backend folder?
- Is there any import in app.py that points to a path that no longer exists
  now that it has moved to sandbox_backend?
- If any import path is broken — show the exact broken line and the fix.

---

## VERDICT

If both checks pass: output exactly "✅ READY TO RUN — python app.py"
If anything is broken: show only the specific lines to fix, nothing else.

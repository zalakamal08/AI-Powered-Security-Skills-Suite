---
name: agentic-ai-red-teaming
description: Red-team an autonomous AI agent (plans, holds memory, calls tools, acts with delegated authority) against the OWASP Agentic AI Top 10 (ASI01-ASI10) — goal hijacking, tool misuse, privilege abuse, unexpected code execution, cascading failures, and rogue-agent persistence. Use for copilots, autonomous coding agents, or any multi-step AI system with real-world side effects.
---

# Agentic AI Red Teaming

Agentic systems add a new attack surface beyond a plain chatbot: they **plan across turns, retain memory, hold credentials, and take actions with real side effects**. The guiding defensive principle from OWASP is "Least Agency" — an agent should get the minimum autonomy required for its task. Your job is to find where that principle was skipped. This skill assumes [[prompt-injection]] and [[mcp-server-security]] as prerequisites/companions — most agentic findings are those techniques chained into an action.

## OWASP Top 10 for Agentic Applications (ASI01–ASI10) — test each

- **ASI01 Agent Goal Hijack**: can injected content (a document, tool result, or another agent's message) change what the agent is trying to accomplish mid-task, not just what it says?
- **ASI02 Tool Misuse and Exploitation**: does the agent call tools outside the intent of the current task when nudged (e.g., a "summarize this email" task that ends up sending one)? See [[mcp-server-security]] for the tool-layer specifics.
- **ASI03 Identity and Privilege Abuse**: does the agent inherit a human's or service account's full permission set rather than a scoped one? Can it be tricked into acting *as* a higher-privileged identity it has access to?
- **ASI04 Agentic Supply Chain Vulnerabilities**: are third-party tools/plugins/sub-agents vetted, version-pinned, and scoped — or pulled in dynamically and trusted by default?
- **ASI05 Unexpected Code Execution**: for coding/dev agents, can generated or agent-modified code execute unintended actions (deleting data, modifying CI, calling out to the network) beyond the stated task? Test destructive-command guardrails specifically (rm/drop/force-push equivalents) — the Replit agent production-database deletion during a code freeze is the canonical real-world case.
- **ASI06 Context Management and Retrieval Manipulation**: can retrieved/RAG context be poisoned to bias the agent's decisions without ever "instructing" it directly?
- **ASI07 Insecure Inter-Agent Communication**: in multi-agent setups, can one agent's output manipulate another agent that trusts it by default? Test whether inter-agent messages get the same scrutiny as user input (usually they don't).
- **ASI08 Cascading Failures**: does one compromised or malfunctioning agent/tool propagate errors or malicious instructions downstream through a pipeline of agents, amplifying impact?
- **ASI09 Human-Agent Trust Exploitation**: can the agent be made to present false confidence, fabricated citations, or misleading action summaries that cause a human approver to rubber-stamp something harmful?
- **ASI10 Rogue Agents**: once compromised or drifted off-task, does the agent keep operating with its original permissions — is there any runtime kill-switch, anomaly detection, or session re-authorization, or does it run unchecked until the task ends?

## Methodology

1. Map the agent's full capability surface: every tool, every credential/identity it can act as, every other agent/system it talks to, and what persists across turns (memory, long-running sessions).
2. For each capability, ask: what's the worst action reachable from here, and what's the minimum trigger (one injected doc? one malicious tool response? one social-engineered instruction?) to reach it.
3. Test destructive/irreversible actions specifically under adversarial and error conditions — not just the happy path an operator demoed.
4. Test whether a human is actually in the loop for consequential actions, or whether "human approval" is a UI step the agent can route around (e.g., auto-approve on timeout, or an approval prompt whose text the agent itself generates and can bias).
5. Test persistence: after a single successful manipulation, does the agent's compromised state/goal survive into later turns or later sessions (memory poisoning), or is it scoped to one exchange?

## Validation

- Every finding needs an actual observed action (tool call executed, file changed, request sent) — not the agent merely narrating intent to do something. Re-run to confirm reproducibility, since agent planning has real run-to-run variance.
- Distinguish "the agent said something wrong" (a jailbreak/quality issue, see [[llm-jailbreaking]]) from "the agent did something wrong" (an agentic-security issue) — the latter is what this skill is scoped to.

## Severity guidance

- **Critical**: goal hijack or privilege abuse leading to irreversible real-world impact (data deletion, unauthorized financial/production action) with no human checkpoint.
- **High**: reliable tool misuse or cross-agent manipulation with a human-approval step that can be bypassed or biased.
- **Medium**: capability exists but requires unrealistic setup, or impact is contained to reversible/low-value actions.

## References

- OWASP Top 10 for Agentic Applications 2026 (genai.owasp.org, published 2025-12-09)
- Real-world incidents cited by OWASP: EchoLeak (CVE-2025-32711), Amazon Q coding-assistant compromise, Replit agent production-database deletion

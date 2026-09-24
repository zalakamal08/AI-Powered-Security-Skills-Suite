# AI-Powered-Security-Skills-Suite

A collection of AI-agent skills for cybersecurity work — reconnaissance, vulnerability analysis, mobile app assessment, LLM/AI security testing, and reporting — designed to be used with Claude and similar AI coding agents.

## What's inside

Skills are organized by security domain, one folder per skill (`skills/<skill-name>/SKILL.md`). Each skill packages the methodology, technique catalog, validation criteria, and severity guidance an AI agent needs to carry out a specific security task consistently — not just a checklist, but how to avoid false positives on each one. Skills cross-reference each other where techniques chain (e.g. injection → data exfiltration → agentic impact).

### AI / LLM Security

| Skill | Covers |
|---|---|
| [`llm-jailbreaking`](skills/llm-jailbreaking) | Guardrail/safety-alignment bypass — DAN/persona, encoding, Crescendo, Echo Chamber, many-shot, Skeleton Key |
| [`prompt-injection`](skills/prompt-injection) | Direct & indirect prompt injection, ASCII/Unicode smuggling, system prompt extraction (OWASP LLM01/LLM07) |
| [`ai-data-exfiltration`](skills/ai-data-exfiltration) | Turning injection into data loss — markdown/image auto-render exfil, tool-call exfil, OOB validation |
| [`mcp-server-security`](skills/mcp-server-security) | MCP tool-poisoning, rug-pull tool redefinition, permission scope, credential aggregation risk |
| [`agentic-ai-red-teaming`](skills/agentic-ai-red-teaming) | OWASP Agentic AI Top 10 (ASI01–ASI10) — goal hijack, tool misuse, privilege abuse, cascading failures |

### Web & API Security

| Skill | Covers |
|---|---|
| [`access-control-testing`](skills/access-control-testing) | IDOR/BOLA, horizontal & vertical privilege escalation, forced browsing, multi-tenant isolation (OWASP A01) |
| [`ssrf-testing`](skills/ssrf-testing) | Basic/blind SSRF, cloud metadata (IMDS), allowlist & parser-confusion bypass, OOB confirmation |
| [`sql-injection`](skills/sql-injection) | Error/UNION/boolean/time-based blind, second-order, NoSQL, WAF evasion |
| [`xss-testing`](skills/xss-testing) | Reflected, stored, DOM, mutation XSS, CSP evaluation & bypass, XSS→ATO |
| [`command-injection`](skills/command-injection) | OS command & code injection, blind/OOB confirmation, argument injection, metacharacter bypass |
| [`ssti-testing`](skills/ssti-testing) | Template injection — polyglot detection, engine fingerprinting, per-engine RCE (Jinja2/Twig/Freemarker/…) |
| [`xxe-testing`](skills/xxe-testing) | XML external entity — file disclosure, SSRF, blind OOB exfiltration, SVG/OOXML/XInclude vectors |
| [`path-traversal-lfi`](skills/path-traversal-lfi) | Directory traversal, LFI/RFI, PHP wrappers/filter chains/log poisoning → RCE |
| [`authentication-testing`](skills/authentication-testing) | Login/session/MFA/reset flaws + JWT attacks (alg=none, weak secret, alg confusion) — with lockout-safety rules |
| [`file-upload-testing`](skills/file-upload-testing) | Content-type/extension/magic-byte bypass, web-shell RCE, stored XSS, unsafe retrieval |
| [`api-security-testing`](skills/api-security-testing) | OWASP API Top 10 2023 — BOLA/BFLA, mass assignment, excessive exposure, GraphQL, business-flow abuse |
| [`business-logic-testing`](skills/business-logic-testing) | Workflow/step bypass, price/quantity tampering, race conditions (TOCTOU), incentive abuse (OWASP A06) |

More skills get added as the toolkit grows — only what's proven useful in real assessments makes the cut.

## Responsible use

Every skill leads with rules of engagement and validation criteria. Use only against systems you are explicitly authorized to test. Findings must be reproducible and evidence-backed (OOB callbacks, two-account proofs, captured output) — the skills are written to avoid false positives, not to maximize noise.

## Usage

Skills in this repo are meant to be loaded into an AI agent (e.g., Claude Code) to assist with authorized security assessments, red teaming, and research. Use only against systems you are authorized to test.

---
Built by [@zalakamal08](https://github.com/zalakamal08)

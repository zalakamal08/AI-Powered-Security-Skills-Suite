---
name: ssrf-testing
description: Test applications for Server-Side Request Forgery — coercing the server to make attacker-controlled requests to internal services, cloud metadata endpoints, or arbitrary URLs. Covers basic/blind SSRF, filter/allowlist bypass, cloud metadata (IMDS) targeting, and OOB confirmation. OWASP A01:2025 and API7:2023. Use wherever the server fetches a URL, imports from a link, renders a webhook, or processes a user-supplied address.
---

# SSRF Testing

SSRF makes the *server* issue a request the attacker chooses — pivoting into internal networks the attacker can't reach directly, reading cloud metadata, or hitting internal admin panels. In 2025 OWASP folded SSRF into A01 (Broken Access Control) and it's API7 in the API Top 10. Because the server has network position and often ambient credentials, SSRF frequently escalates to sensitive-data disclosure or RCE.

## Find the injection points

Anywhere the server takes a URL, hostname, IP, or file path and fetches/renders it:
- Webhooks, callback URLs, "import from URL", "fetch avatar from URL", PDF/screenshot/thumbnail generators, link previews/unfurlers.
- XML/SVG/HTML parsers (SVG `<image href>`, see also [[xxe-testing]]), document converters, headless-browser renderers.
- URL params that look like proxies (`?url=`, `?next=`, `?dest=`, `?feed=`, `?image=`, `?callback=`, `?domain=`).
- API integrations where you supply a base URL / API endpoint.
- File-path params that accept `file://`, `gopher://`, `dict://`, etc.

## Core tests

1. **Basic / in-band**: point the param at an attacker-controlled server; confirm the server connects (check your logs). Then try internal targets: `http://127.0.0.1:<port>`, `http://localhost/admin`, common internal ranges `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
2. **Blind SSRF**: no response reflected — use an OOB listener (Burp Collaborator / interactsh). A DNS or HTTP callback confirms the server resolved/fetched your URL. This is the primary confirmation method; treat a landed callback as the proof.
3. **Cloud metadata (IMDS)** — highest impact, always test:
   - AWS IMDSv1: `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (steal role creds). Note IMDSv2 requires a PUT token first — if you can't chain that, report the attempt/impact.
   - GCP: `http://metadata.google.internal/computeMetadata/v1/` (needs `Metadata-Flavor: Google` header — test if you control headers).
   - Azure: `http://169.254.169.254/metadata/instance?api-version=2021-02-01` (needs `Metadata: true`).
4. **Port/service scan via SSRF**: differentiate open/closed/filtered internal ports by response/timing differences.

## Filter & allowlist bypasses

When naive blocklists try to stop `127.0.0.1`/`localhost`/`169.254.169.254`:
- **Alternate IP encodings**: decimal `2130706433`, octal `0177.0.0.1`, hex `0x7f000001`, IPv6 `[::1]`, `[::ffff:127.0.0.1]`, short forms `127.1`, `0.0.0.0`.
- **DNS tricks**: attacker domain resolving to internal IP; `localtest.me`/`nip.io`-style wildcard DNS; **DNS rebinding** (TOCTOU — record passes the allowlist check, then re-resolves to internal IP on fetch).
- **URL parser confusion**: `http://expected.com@169.254.169.254/`, `http://169.254.169.254#expected.com`, `http://expected.com.attacker.com`, backslashes, embedded `@`, and unicode/idn tricks that the validator and the HTTP client parse differently.
- **Redirect bypass**: allowlisted URL that 30x-redirects to an internal target (test whether the client follows redirects).
- **Scheme abuse**: `file://`, `gopher://` (craft raw TCP → can hit Redis/SMTP for RCE), `dict://`, `ftp://`.

## Escalation

- SSRF → cloud creds (IMDS) → account takeover of the whole environment.
- SSRF → internal admin/actuator endpoints (Spring `/actuator`, `/env`), Redis/Memcached (via gopher), Docker/K8s API.
- SSRF → LFI via `file://` (see [[path-traversal-lfi]]); SSRF via XML parsers (see [[xxe-testing]]).

## Validation — avoid false positives

- Blind SSRF requires a **real received callback** (DNS or HTTP) on your listener — a slow/erroring response alone is not proof.
- For IMDS/internal reads, capture the actual returned content (or a differential timing signal) as evidence, not just a 200.
- Rule out that the "internal" response is really an error page or a WAF interstitial.

## Severity guidance

- **Critical**: SSRF reaching cloud metadata and returning usable credentials, or reaching an internal service that yields RCE.
- **High**: confirmed access to internal-only services/data, or reliable internal port scanning.
- **Medium**: blind SSRF with OOB callback but no demonstrated internal reach yet.

## References

- OWASP Top 10 2025 — A01 (SSRF folded in); OWASP API Top 10 2023 — API7 SSRF
- OWASP WSTG v4.2 — 4.7.19 Testing for SSRF

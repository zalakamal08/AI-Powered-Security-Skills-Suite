---
name: api-security-testing
description: Test REST/GraphQL APIs against the OWASP API Security Top 10 2023 — BOLA/BFLA authorization, broken authentication, property-level authorization (mass assignment / excessive data exposure), resource consumption, business-flow abuse, SSRF, and improper inventory. Use for any backend API, mobile app backend, or GraphQL endpoint.
---

# API Security Testing

APIs shift the biggest risks to **authorization at the object and function level** and to **what data the API over-returns or over-accepts**. This skill maps the OWASP API Security Top 10 2023; several items have dedicated companion skills — use them for depth.

## Setup

- Get the API surface: OpenAPI/Swagger spec, GraphQL schema (introspection), mobile-app traffic, or crawl + JS analysis for endpoints.
- Two accounts (and two tenants) minimum, plus one per role — same rationale as [[access-control-testing]].
- Proxy all traffic; keep per-identity request collections for replay.

## OWASP API Top 10 2023 — test each

- **API1 BOLA (Broken Object Level Authorization)** — #1: swap object IDs across users/tenants on every object-bearing endpoint (read/write/delete/create-as-other). Full method in [[access-control-testing]].
- **API2 Broken Authentication**: weak token/JWT handling, credential-stuffing exposure, no rotation — see [[authentication-testing]] (respect lockout/rate-limit rules).
- **API3 Broken Object Property Level Authorization** — combines two classic bugs:
  - **Mass assignment**: add unexpected fields to a write request (`"role":"admin"`, `"isVerified":true`, `"balance":9999`, `"userId":<other>`) and see if the server binds them → privilege/state escalation.
  - **Excessive data exposure**: the API returns more fields than the UI shows (other users' PII, internal flags, password hashes). Inspect raw JSON, not the rendered page.
- **API4 Unrestricted Resource Consumption**: missing pagination limits, large `limit`/`page_size`, expensive queries, no rate limiting — demonstrate with a *bounded* test (don't DoS); note cost-amplification (email/SMS/compute) flows.
- **API5 BFLA (Broken Function Level Authorization)**: call admin/privileged functions as a low-priv user; method-swap (`GET`→`PUT`/`DELETE`), hit `/admin/*` and internal management endpoints — see [[access-control-testing]].
- **API6 Unrestricted Access to Sensitive Business Flows**: automate a business flow (bulk purchase, ticket scalping, referral/coupon abuse, mass account creation) that should have anti-automation — see [[business-logic-testing]].
- **API7 SSRF**: URL-accepting API params → internal/metadata reach — see [[ssrf-testing]].
- **API8 Security Misconfiguration**: verbose errors/stack traces, missing security headers, permissive CORS (reflected `Origin` + `Access-Control-Allow-Credentials: true`), unnecessary HTTP methods, debug endpoints.
- **API9 Improper Inventory Management**: undocumented/old API versions (`/v1` vs `/v2`), staging/debug hosts, deprecated endpoints still live with weaker controls.
- **API10 Unsafe Consumption of APIs**: the target trusts a third-party API's data insufficiently (injection/SSRF via upstream responses).

## GraphQL specifics

- **Introspection** enabled → dump the full schema (`__schema`) to map every query/mutation and type.
- **Authorization per resolver**: BOLA/BFLA apply per field/resolver — test object access and mutations individually.
- **Batching/aliasing abuse**: send many aliased queries in one request to bypass rate limits (brute-force OTP/login via batching — mind [[authentication-testing]] safety rules) or to amplify cost.
- **Deep/nested queries**: recursive relationships → DoS (test bounded).
- **Injection** through arguments into downstream stores (see [[sql-injection]]).

## REST specifics

- Enumerate methods per endpoint (`OPTIONS`, then test each). Test content-type switching (JSON↔XML→[[xxe-testing]]).
- Parameter pollution (`?id=1&id=2`), wrapping IDs in arrays/objects to bypass authz checks.
- Header-based auth/tenant selectors (`X-User-Id`, `X-Tenant-Id`) that the server trusts.

## Validation — avoid false positives

- BOLA/BFLA: prove with two real accounts that you accessed the *other* principal's data/function (see [[access-control-testing]] validation).
- Mass assignment: confirm the injected field actually changed server state (re-fetch the object), not just that the request returned 200.
- Excessive data exposure: confirm the extra fields contain real sensitive data, not nulls/placeholders.
- Resource-consumption findings: demonstrate the gap with a small bounded test; never actually degrade the service.

## Severity guidance

- **Critical**: unauth/cross-tenant BOLA, mass-assignment privilege escalation, admin BFLA, SSRF-to-metadata.
- **High**: authenticated BOLA on PII, excessive exposure of sensitive fields, business-flow abuse with financial impact.
- **Medium**: misconfig (CORS, verbose errors), missing rate limits without demonstrated abuse.

## References

- OWASP API Security Top 10 2023 (owasp.org/API-Security)
- OWASP WSTG v4.2 — API testing; PortSwigger — API testing, GraphQL

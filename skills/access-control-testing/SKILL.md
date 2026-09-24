---
name: access-control-testing
description: Test web/API applications for broken access control — IDOR/BOLA (object-level), horizontal and vertical privilege escalation, function-level authorization gaps, forced browsing, and multi-tenant isolation failures. OWASP A01:2025 (#1) and API1:2023 BOLA. Use whenever the target has authenticated users, roles, or per-object ownership.
---

# Access Control Testing

Broken access control is the #1 risk in OWASP Top 10 2025 (A01) and #1 in the API Top 10 2023 (BOLA). The core test is always the same: **can principal A reach an object or function that should belong only to principal B (horizontal) or to a higher role (vertical)?** You cannot test this properly with one account — get at least two users, ideally two users per role tier, before you start.

## Prerequisites

- **Two low-priv accounts** (User A, User B) to prove horizontal / cross-user access.
- **One account per privilege tier** (e.g., user, manager, admin) to prove vertical escalation.
- In multi-tenant apps, **two accounts in different tenants/orgs** to prove tenant isolation breaks.
- An intercepting proxy (Burp/ZAP) with request history for both sessions so you can replay A's requests with B's identifiers, and vice versa.

## IDOR / BOLA — object-level authorization

IDOR is the mechanism (a direct, guessable object reference); BOLA is the root cause (server never checks the requester owns the object). Test all five verbs of access against another principal's object:

1. **Read**: capture a request that returns User A's object (`/api/orders/1001`, `?userId=A`), replay it in User B's session with A's object ID. If B gets A's data → BOLA read.
2. **Write/update**: replay A's update request in B's session targeting A's object — does it mutate A's data?
3. **Delete**: same, for delete endpoints.
4. **Create-as-other**: can you set an `ownerId`/`userId`/`tenantId` field in a create request to assign the object to another principal?
5. **Cross-tenant**: substitute another tenant's org/account ID in path, query, body, or header (`X-Tenant-Id`, `X-Account-Id`).

Identifier hunting tips:
- Not just numeric IDs — test UUIDs (often leaked elsewhere and still not authz-checked), slugs, hashids, base64-wrapped IDs, and composite keys.
- Look for IDs in **all locations**: path, query string, body (JSON/form), custom headers, cookies, and JWT claims.
- Predictable IDs (sequential ints) make it worse but a "random" UUID that's exposed in another response is still exploitable — authz, not obscurity, is the control.

## Privilege escalation

- **Horizontal**: reach a same-tier user's data/functions (this is BOLA above).
- **Vertical**: reach admin/manager functionality as a normal user.
  - Capture admin-only requests (from an admin session or from JS/route tables — see below) and replay them in a low-priv session.
  - Test **function-level** authz (OWASP API5 BFLA): `POST /api/admin/users`, `DELETE /api/users/{id}`, role-change endpoints, `PATCH /api/users/me {"role":"admin"}` (mass-assignment privesc — see [[api-security-testing]]).
  - Method swap: if `GET /admin/x` is blocked, try `POST`, `PUT`, `HEAD`, or override headers (`X-HTTP-Method-Override`).

## Forced browsing & missing function-level checks

- Enumerate admin/privileged endpoints even when the UI hides them — a hidden menu item is not access control. Pull candidate routes from client-side JS bundles and SPA route tables (they often list admin routes with their role guards, mapping privileged surface without needing an admin account).
- Test static/secondary endpoints that bypass the main authz layer: export/report generators, PDF/CSV download URLs, legacy `/v1/` APIs, GraphQL resolvers, direct file/object-store links.

## Common bypass patterns to try

- Path tricks: `/admin` blocked but `/admin/`, `/./admin`, `/%2e/admin`, `/admin..;/`, case changes, or trailing `%00`/`.json` allowed.
- Referer/Origin-based "auth" (trivially spoofable).
- Client-side-only enforcement: the API returns the data and the frontend just hides it.
- Step-skipping in multi-step flows (jump straight to the confirm/fulfil endpoint).

## Validation — avoid false positives

- Confirm with **two real accounts** that the data returned actually belongs to the *other* principal (check a unique attribute you set on B's object), not your own or empty/placeholder data.
- Distinguish "endpoint returns 200 with someone else's data" (real) from "returns 200 but only your own data / an empty object" (not a finding).
- Re-run to be sure a session token didn't silently carry over between the two identities in your proxy.

## Severity guidance

- **Critical**: unauthenticated or cross-tenant access to sensitive data, or vertical escalation to admin.
- **High**: authenticated horizontal IDOR exposing other users' PII/financial data (read or write).
- **Medium**: IDOR on low-sensitivity objects, or function-level gap with limited impact.

## References

- OWASP Top 10 2025 — A01 Broken Access Control (SSRF now folded in here; see [[ssrf-testing]])
- OWASP API Security Top 10 2023 — API1 BOLA, API5 BFLA (see [[api-security-testing]])
- OWASP WSTG v4.2 — 4.5 Authorization Testing

---
name: authentication-testing
description: Test authentication, session management, and JWT handling — credential/login flaws, session lifecycle, MFA/reset/registration weaknesses, and JWT attacks (alg=none, weak secret, algorithm confusion, claim tampering). OWASP A07:2025 Authentication Failures. Use for any login, session, token, password-reset, or SSO flow.
---

# Authentication & Session Testing

Covers OWASP A07:2025 (Authentication Failures) plus session management and JWT. The goal is to break identity assurance: log in as someone else, keep a session you shouldn't, or forge a token.

## SAFETY FIRST — account lockout discipline (read before any login test)

- **Never loop logins against the same account** — one login per test account per run. Repeated attempts trigger lockouts.
- **No brute-force/credential-spraying/fuzzing of login endpoints without explicit written permission**, even with test accounts.
- **Confirm the lockout policy** with the client before any multi-login test; add a **2–3s minimum delay** between auth requests.
- If a lockout occurs, **stop immediately, report, and wait** for accounts to be unlocked. Use client-provided or Yopmail/test accounts only.
- Rate-limit *testing* (the finding "rate limiting is missing") is proven with a small, bounded burst — not by actually locking out or DoSing the endpoint.

## Credential & login flaws

- **User enumeration**: differing responses/timing between valid and invalid usernames on login, registration, and password reset. (Test with your own accounts; don't enumerate real users at scale.)
- **Weak password policy**: register/change to a trivially weak password (bounded, one attempt).
- **Default/known credentials** on admin panels and appliances (single try each).
- **Missing rate limiting / lockout** on login and OTP endpoints — demonstrate with a *small* bounded burst plus the lockout-policy caveat above; do not actually lock the account.
- **Auth bypass**: SQLi in login (`' OR 1=1-- -`, see [[sql-injection]]), NoSQL operator injection, response manipulation (`"success":false`→`true`), forced-browsing past the login (see [[access-control-testing]]).

## Session management

- **Post-login token rotation**: session ID must change on login (else session fixation).
- **Logout / expiry**: session must be invalidated server-side on logout and after idle/absolute timeout — replay the old token after logout.
- **Cookie flags**: `HttpOnly`, `Secure`, `SameSite`; scope/path correctness.
- **Predictable/weak session IDs**: entropy and structure.
- **Concurrent sessions**: whether multiple sessions are allowed and whether a password change kills existing ones.
- **CSRF** on state-changing actions (missing/again-usable tokens) — often tested alongside session.

## Password reset / registration / MFA

- **Reset token**: predictability, expiry, single-use, binding to the user; **host-header poisoning** to send the reset link to an attacker domain; token leakage in Referer.
- **Account takeover via reset**: change email/username mid-flow, or reuse a token across accounts.
- **MFA bypass**: OTP brute-force (bounded!), reuse, missing enforcement on some endpoints, backup-code weaknesses, "remember device" flaws, or skipping the MFA step by direct-requesting the post-MFA endpoint.
- **Registration**: duplicate/overwrite accounts, email verification bypass, role assignment via mass assignment (see [[api-security-testing]]).

## JWT attacks (very high yield — ~30% of audited implementations have a critical flaw)

- **Decode first**: base64-decode header+payload; check for sensitive data (PII) in claims, and note `alg`, `kid`, `iss`, `aud`, `exp`.
- **alg=none**: set header `{"alg":"none"}`, strip the signature — if accepted, forge any identity.
- **Weak HMAC secret**: crack `HS256` offline with a wordlist (hashcat mode 16500) — if cracked, sign arbitrary tokens.
- **Algorithm confusion (RS256→HS256)**: sign with the public key as the HMAC secret when the server doesn't pin the algorithm.
- **Claim tampering**: change `role`/`user_id`/`tenant`/`admin` and see if the signature is actually verified (test alongside alg=none).
- **`kid` injection**: path traversal / SQLi in the `kid` header to control the verification key.
- **Missing validation**: expired (`exp`) tokens accepted, wrong `aud`/`iss` accepted, token from one context reused in another.
- **No revocation**: token still valid after logout/password change.

## Validation — avoid false positives

- Prove auth bypass / forged token by actually accessing a protected resource as the target identity, not just a 200 on the login call.
- For JWT, confirm the server *acts on* the tampered claim (returns the other user's data / admin function), not just that it echoes the token.
- Keep evidence bounded and within the lockout/rate-limit rules above.

## Severity guidance

- **Critical**: auth bypass, account takeover (reset/JWT forge), or admin access.
- **High**: user enumeration + weak lockout enabling practical attack; session not invalidated on logout for sensitive app.
- **Medium**: missing cookie flags, weak password policy, verbose enumeration alone.

## References

- OWASP Top 10 2025 — A07 Authentication Failures
- OWASP WSTG v4.2 — 4.4 Authentication, 4.6 Session Management
- PortSwigger — JWT attacks

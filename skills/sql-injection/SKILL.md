---
name: sql-injection
description: Test applications for SQL injection — error-based, UNION-based, boolean and time-based blind, second-order, and NoSQL injection, plus WAF-evasion techniques. OWASP A05:2025 Injection. Use for any parameter that reaches a database query (search, filters, login, IDs, sort/order, headers, JSON fields).
---

# SQL Injection

SQL injection lets attacker input alter the structure of a database query — reading, modifying, or exfiltrating data, sometimes reaching the OS. It's part of OWASP A05:2025 (Injection). Methodology: **detect → determine type/DBMS → extract/escalate**, always confirming safely before pulling real data.

## Find injection points

Every parameter that could reach a query: URL/query params, POST body (form + JSON), path segments, cookies, `User-Agent`/`Referer`/`X-Forwarded-For` (logged/queried), `ORDER BY`/`sort` columns, `LIMIT`, search filters, and multi-tenant selectors. Don't forget **second-order**: input stored now (e.g., profile name) and used unsafely in a query later.

## Detection

1. **Probe with breaking characters**: `'`, `"`, `` ` ``, `\`, `;`, `)`. A DB error, a 500, or a content/behavior change signals query interference.
2. **Prove logic control** (boolean): `' AND 1=1-- -` vs `' AND 1=2-- -` — different responses = injectable. For numeric contexts drop the quote.
3. **Confirm execution** rather than an incidental error — the true/false pair must flip a genuine query result, not just trigger a generic error both times.

## Exploitation by type

- **Error-based**: coax the DBMS to leak data inside error messages (e.g., `extractvalue`, `updatexml` on MySQL; `CAST`/conversion errors on MSSQL/Postgres). Fastest when errors are shown.
- **UNION-based**: find column count (`ORDER BY n` until error, or `UNION SELECT NULL,NULL...`), find a reflected/text column, then select `@@version`, `current_user`, schema tables. Requires results reflected in the response.
- **Boolean-based blind**: infer data one condition at a time from a true/false response difference (`... AND SUBSTRING(...,1,1)='a'`).
- **Time-based blind**: no visible difference — use conditional delays: MySQL `SLEEP(5)`, Postgres `pg_sleep(5)`, MSSQL `WAITFOR DELAY '0:0:5'`, Oracle heavy queries / `DBMS_LOCK.SLEEP`. Confirm the delay tracks the condition (and repeats) to rule out network jitter.
- **OOB**: when in-band and time channels are blocked, force a DNS/HTTP callback (MSSQL `xp_dirtree`/`UNC`, Oracle `UTL_HTTP`, MySQL `LOAD_FILE`+UNC on Windows) to an interactsh/Collaborator listener.

## DBMS fingerprinting

Use version funcs (`@@version`, `version()`, `banner`), string concat differences (`||` vs `CONCAT` vs `+`), comment styles (`-- `, `#`, `/* */`), and error signatures to pick the right payloads for the target engine.

## WAF / filter evasion (only against the client's own WAF, in scope)

- Case/comment obfuscation: `SeLeCt`, `SEL/**/ECT`, inline `/*!50000UNION*/`.
- Function synonyms that dodge signatures: `substring()`→`mid()`/`substr()`, `sleep()`→`benchmark()`, `ascii()`→`ord()`.
- Encoding: URL/double-URL, unicode, hex-encoded strings; whitespace alternatives (`/**/`, `%09`, `%0a`, `+`).
- Logic rewrites: `OR 1=1`→`OR 2>1`, `=`→`LIKE`/`BETWEEN`.
- Note: manual testing behind the WAF is the reliable path — automated scanners get filtered.

## NoSQL injection

For MongoDB-style backends test operator injection: `{"user":{"$ne":null},"pass":{"$ne":null}}`, `$gt`, `$regex`, and JS `$where` payloads; in query strings `user[$ne]=`. Auth bypass and blind extraction work analogously to classic SQLi.

## Tooling note

Manual confirmation first; `sqlmap` is appropriate for extraction once a point is confirmed and in scope — throttle it and avoid destructive `--os-shell`/stacked-query tests without explicit permission. Never run mass automated injection against production without sign-off.

## Validation — avoid false positives

- Time-based: repeat with different delay values (0s, 5s, 10s) and confirm the response time scales with the payload, not the network.
- Boolean: confirm the true/false responses differ on a *data-dependent* condition, not a syntax error that happens to differ.
- Distinguish a generic 500/WAF block from actual query manipulation before reporting.

## Severity guidance

- **Critical**: data extraction of sensitive tables, auth bypass, or RCE via stacked queries / `xp_cmdshell` / file write.
- **High**: confirmed blind extraction of any user data.
- **Medium**: injectable point proven but access limited (e.g., only booleans, no sensitive data reachable).

## References

- OWASP Top 10 2025 — A05 Injection
- OWASP WSTG v4.2 — 4.7.5 Testing for SQL Injection

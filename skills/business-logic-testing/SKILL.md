---
name: business-logic-testing
description: Test for business-logic and design flaws that scanners miss — workflow/step bypass, parameter/price/quantity tampering, race conditions (TOCTOU), negative/overflow values, coupon/referral abuse, and unrestricted business-flow automation. OWASP A06:2025 Insecure Design and API6:2023. Use once you understand the app's intended rules and money/state-changing flows.
---

# Business Logic Testing

Business-logic flaws are legitimate-looking requests that violate the application's *intended rules* — they rarely trip scanners because nothing is malformed. Maps to OWASP A06:2025 (Insecure Design) and API6:2023 (Unrestricted Access to Sensitive Business Flows). You must first **understand the intended workflow and its assumptions**, then test what happens when you break each assumption.

## Method

1. **Model the flow**: map each multi-step process (checkout, transfer, registration, KYC, refund, subscription) — its steps, required order, and the invariants the developers assumed ("price is set server-side", "you can't skip payment", "quantity ≥ 1", "you own this cart").
2. **For each assumption, break it** and observe whether the server enforces it or trusts the client.

## Core test classes

**Parameter / value tampering**
- **Price/amount**: modify price, subtotal, currency, or discount fields in the request — is the price recomputed server-side or trusted from the client?
- **Quantity/negative/overflow**: `-1`, `0`, huge values, decimals where integers expected → negative totals, free items, refund-to-attacker, integer overflow.
- **State/status fields**: set `status=paid`, `approved=true`, `role=admin` (mass assignment — see [[api-security-testing]]).
- **ID/ownership**: change `cartId`/`accountId`/`userId` to act on another's resource (see [[access-control-testing]]).

**Workflow / step bypass**
- Skip steps: go straight to the confirmation/fulfilment endpoint without payment or validation.
- Repeat steps that should be once-only (apply a coupon N times, redeem a one-time token twice).
- Reorder steps; resume an abandoned flow with modified data; replay a completed transaction.

**Race conditions (TOCTOU)**
- Fire concurrent requests to exploit check-then-act gaps: redeem a single-use coupon/gift card multiple times, withdraw/transfer beyond balance, double-spend, over-book limited stock, bypass "one vote/action per user". Use parallel/last-byte-sync requests. Confirm by observing the invariant broken (e.g., balance went negative, coupon used twice).

**Discount / referral / incentive abuse**
- Stack coupons, self-referral, create accounts to farm signup credits, abuse loyalty points, exploit rounding.

**Unrestricted business-flow automation (API6)**
- Automate a sensitive flow that should have anti-automation (bulk purchase/scalping, mass messaging, mass account creation, data scraping). Demonstrate the missing control with a *small bounded* run — do not actually flood the service or create mass real-world effects.

**Constraint & validation gaps**
- Client-side-only limits (min/max, date ranges, age gates) not re-checked server-side.
- Time-based logic: back/forward-dating, expiry bypass, timezone confusion.

## Validation — avoid false positives

- Confirm the flaw produced a real, unintended outcome (money/state actually changed in the attacker's favor, invariant actually broken) — not just a 200 response.
- Race conditions: reproduce the double-effect and show the end state (e.g., two redemptions recorded); a single lucky timing is not enough — repeat.
- Keep tests bounded and reversible; for financial/irreversible actions, prove the smallest case and stop (don't repeat for effect).

## Severity guidance

- **Critical**: direct financial loss, free goods/services at scale, or trust-boundary bypass with broad impact.
- **High**: reliable single-instance monetary/logic abuse (one free order, one balance overrun).
- **Medium**: abuse requiring effort or yielding limited value; missing anti-automation without demonstrated large impact.

## References

- OWASP Top 10 2025 — A06 Insecure Design
- OWASP API Security Top 10 2023 — API6 Unrestricted Access to Sensitive Business Flows
- OWASP WSTG v4.2 — 4.10 Business Logic Testing

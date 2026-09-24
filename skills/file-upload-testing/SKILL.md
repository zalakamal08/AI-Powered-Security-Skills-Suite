---
name: file-upload-testing
description: Test file-upload features for unrestricted/dangerous upload leading to RCE, stored XSS, SSRF, or path traversal — content-type/extension/magic-byte bypass, dangerous file types, and unsafe storage/retrieval. OWASP A05:2025 / A01:2025. Use for any avatar, document, import, attachment, or media upload.
---

# File Upload Testing

Upload flaws range from full RCE (uploading a web shell into a served, executable directory) down to stored XSS and SSRF. The two questions that decide severity: **what does the server let you upload**, and **what happens when the file is stored/retrieved/rendered**. Test both the validation and the retrieval path.

## Recon the control

- What extensions/content-types are accepted? Where are files stored, and are they served from a web-executable path or a separate/static host?
- Is the filename preserved (traversal risk) or randomized? Is the file processed (image resize, AV scan, conversion)?
- Is retrieval authenticated and is the served `Content-Type`/`Content-Disposition` safe?

## Validation-bypass techniques

**Extension checks:**
- Alternate executable extensions: `.php5`, `.phtml`, `.pht`, `.phar`, `.jsp`, `.jspx`, `.aspx`, `.asmx`, `.cshtml`.
- Case tricks: `.pHp`, `.PHP`.
- Double extensions: `shell.php.jpg`, `shell.jpg.php`; trailing chars `shell.php.`, `shell.php%20`, `shell.php%00.jpg` (null byte on legacy stacks).
- Config-file uploads: `.htaccess`/`web.config` to make the server execute an otherwise-benign extension.

**Content-Type checks:** set `Content-Type: image/png` on the multipart part while the body is a script.

**Magic-byte / content sniffing:** prepend valid file signatures (e.g., `GIF89a;` then PHP, or a real image header) so the file passes magic-number validation but still executes — a polyglot (valid image + script).

**Image processing bypass:** embed payloads in EXIF/metadata; use image-parser CVEs (ImageMagick/ffmpeg) for command injection (see [[command-injection]]); SVG uploads for stored XSS/XXE (see [[xss-testing]], [[xxe-testing]]).

## Impact paths to prove

- **RCE**: upload a web shell to an executable, web-served path → request it → run a benign `id`/OOB callback. Highest severity.
- **Stored XSS**: upload HTML/SVG served inline with a scriptable `Content-Type` → fires in a viewer's browser.
- **Path traversal on store**: filename `../../var/www/html/shell.php` writes outside the upload dir (see [[path-traversal-lfi]]).
- **SSRF/XXE**: SVG/XML/OOXML uploads processed server-side (see [[ssrf-testing]], [[xxe-testing]]).
- **DoS / logic**: oversized files, zip bombs (test cautiously, with permission), or overwriting other users' files via predictable paths.
- **Client-side / AV**: malicious content served to other users; missing scanning.

## Retrieval-side checks

- Is the uploaded file served with `Content-Disposition: attachment` and a safe content type, or rendered inline (XSS risk)?
- Is access to other users' uploaded files authorized (IDOR on the download URL — see [[access-control-testing]])?
- Can you guess/enumerate other users' file URLs?

## Validation — avoid false positives

- For RCE, actually execute the uploaded file (benign proof/OOB) — a successful upload alone is not RCE if the file is never served or never executed.
- Confirm stored XSS fires in a rendered context, not just that the file was stored.
- Verify the file is reachable/served before claiming impact.

## Severity guidance

- **Critical**: upload → RCE (web shell in executable path).
- **High**: stored XSS reaching other users, or path traversal writing to sensitive locations.
- **Medium**: dangerous file type accepted but not executed/served dangerously; SSRF/XXE via processing.
- **Low**: missing type/size validation with no demonstrated impact.

## References

- OWASP Top 10 2025 — A05 / A01
- OWASP WSTG v4.2 — 4.10 Business Logic / File Upload; 4.7 Input Validation

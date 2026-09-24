---
name: mcp-server-security
description: Security-review an MCP (Model Context Protocol) server or client integration — tool-poisoning, rug-pull tool redefinition, over-broad permission grants, credential aggregation risk, and missing output sanitization. Use when a target exposes or consumes MCP tools/servers for an AI agent.
---

# MCP Server Security

MCP servers sit between an AI agent and real systems (filesystems, shells, databases, SaaS APIs), so a compromised or malicious MCP server has **ambient authority** over everything the agent is allowed to touch. Treat every MCP server as untrusted until proven otherwise, including ones from public registries.

## Core threats

### Tool poisoning
A malicious or compromised server can embed hidden instructions inside a tool's **description/schema** (not just its output) — text the agent reads as part of tool selection but a human reviewing the UI never sees. Test:
- Fetch every registered tool's full description/schema and read it for embedded imperative instructions ("always also call X", "never mention Y to the user"), not just its stated purpose.
- Check whether descriptions reference other tools by name to manipulate call ordering or suppress logging.

### Rug-pull / mutable tool definitions
A server can change a tool's behavior or description *after* a user approved it once. Test:
- Whether the client re-validates/re-displays tool definitions on every call or only at first connection/consent.
- Whether tool definitions are pinned/hashed anywhere, or silently trusted on every session.

### Output sanitization gaps
Tool call results flow back into the model's context as if they were trusted data. Test:
- Whether a tool response containing prompt-injection payloads (see [[prompt-injection]]) is sanitized before re-entering context, or passed through raw.
- Whether error messages/stack traces from a tool leak internal paths, credentials, or infrastructure details into the model context (and potentially back to the user).

### Over-broad permission grants
- Enumerate what the server can actually reach (filesystem root, shell, full-account API tokens) vs. what the agent's task requires. Flag "all-access" configs — least privilege is the exception in most current deployments, not the norm.
- Test whether the agent can be steered (via injection or direct request) to use a tool outside its intended task scope — e.g., a "read this one file" tool that actually accepts arbitrary paths (path traversal into an unintended filesystem scope).

### Credential aggregation / single point of failure
- MCP servers often hold long-lived credentials for multiple downstream services. Confirm how those credentials are stored, whether the server enforces per-client authentication/authorization (many ship with none by default), and what a full server compromise would expose.
- Check transport security: local stdio servers trust anything on the same machine; remote/SSE servers need real auth (OAuth/token), not just network-location trust.

### Supply chain
- For third-party/community MCP servers: check publisher reputation, version pinning, and whether the install path allows silent auto-updates (a clean server today can turn hostile in a later release — the classic rug-pull).

## Methodology

1. Inventory every MCP server the target agent connects to and every tool each one exposes (name, description, input schema, declared permissions).
2. Diff tool descriptions across sessions/versions if possible — flag any that changed without new user consent.
3. For each tool, test with adversarial input: oversized payloads, path traversal, command-injection-style strings, and injected instructions in any free-text field.
4. Confirm whether tool responses are sanitized before re-entering the model's context (use a response containing a fake instruction and see if the agent obeys it).
5. Map credential/permission scope per server; flag any server holding more access than its declared purpose needs.

## Severity guidance

- **Critical**: unauthenticated remote MCP server, or a tool-poisoning/rug-pull path that lets an attacker pivot to real command execution or credential theft.
- **High**: tool response content isn't sanitized before re-entering agent context, enabling injection chains into other tools.
- **Medium**: over-broad permission grant with no demonstrated exploit path yet; missing tool-definition pinning.

## References

- Cloud Security Alliance, "Agentic MCP Security Best Practices Guide"
- "MCP-38: A Comprehensive Threat Taxonomy for Model Context Protocol Systems" (arXiv 2603.18063)
- NSA/CISA, "Model Context Protocol (MCP) Security: Design Considerations" (media.defense.gov, 2026)
- CVE-2025-6514 (MCP trust-boundary failure)

# CLAUDE.md — docker-traefik Project Context

## Project Overview

This repository manages a multi-host Docker + Traefik reverse-proxy infrastructure across:
- **hs** — Home Server (Proxmox LXC, Ubuntu 22.04)
- **mds** — Media/Database Server (Proxmox LXC, Ubuntu 22.04)
- **ws** — Web Server (DigitalOcean VPS, Ubuntu 22.04)
- **ds918** — Synology DS918+ NAS
- **dns** — DNS/AdBlock (Raspberry Pi 4B)

Compose files are split per host under `compose/$HOSTNAME/`. Secrets are file-based under `secrets/`. Shared config lives in `shared/`.

---

## Claude Code — Available Skills

Invoke with `/skill-name` in any session on this repo:

| Skill | Purpose |
|-------|---------|
| `/session-start-hook` | Set up startup hooks for Claude Code on the web |
| `/update-config` | Configure settings.json — hooks, permissions, env vars |
| `/keybindings-help` | Customize keyboard shortcuts / keybindings.json |
| `/verify` | Run the app and confirm a change works in practice |
| `/code-review` | Review current diff for bugs (supports `--fix` and inline PR comments) |
| `/fewer-permission-prompts` | Scan transcripts and add allowlist entries to reduce prompts |
| `/loop` | Run a command on a recurring interval |
| `/claude-api` | Build/debug/optimize Anthropic SDK apps |
| `/run` | Launch the project app to test a change live |
| `/init` | Initialize a new CLAUDE.md with codebase docs |
| `/review` | Review a pull request |
| `/security-review` | Full security review of pending branch changes |
| `/simplify` | Review current diff and apply fixes |

## GitHub Access

- **Repo in scope:** `crdollar/docker-traefik` (read/write via GitHub MCP tools)
- Branches, PRs, commits, issues, file contents, CI status are all accessible
- Calls targeting any other repository will be denied

---

## NSA MCP Security Guidelines (Permanent Reference)

**Source:** NSA Cybersecurity Information Sheet — *"Model Context Protocol (MCP): Security Design Considerations for AI-Driven Automation"* (AISC, May 2026, U/OO/6030316-26 PP-26-1834 Ver. 1.0)

These guidelines apply whenever MCP servers, AI agents, or AI-driven automation are added to this infrastructure. Treat this as a standing security checklist for any AI/MCP-adjacent work.

### Threat Categories to Design Against

| Threat | Description |
|--------|-------------|
| **Tool Poisoning** | Malicious tool descriptions in MCP metadata manipulate AI behavior covertly |
| **Rug Pull** | A trusted tool mutates its definition after approval; credentials or behavior change silently |
| **Prompt Injection** | Malicious content in tool responses hijacks agent actions (e.g., exfiltrate repo data via a legitimate tool) |
| **Shadow Servers** | Attacker registers a tool with the same name as a trusted one; agent selects the wrong one |
| **Confused Deputy** | OAuth proxy exploitation — static client IDs + dynamic registration + stored consent cookies allow authorization code theft without fresh user approval |
| **DNS Rebinding** | Malicious website sends requests to localhost MCP servers by exploiting DNS TTL (CVE-2025-66414, CVSS 7.6 in MCP TypeScript SDK) |
| **Credential Aggregation** | One MCP server holds creds for multiple services — single compromise = full blast radius |
| **SSRF** | MCP server tool fetches attacker-controlled URLs → pivots to internal metadata endpoints (AWS IAM, GCP SA tokens) |
| **Privilege Escalation** | Compromised tool gains lateral movement through shared network or over-permissive mounts |

---

### NSA Recommendations — By Role

#### For Operators (Infrastructure / Docker / Traefik)

- **Network isolation**: MCP server containers must not share a network with unrelated services. Use dedicated Docker networks per trust boundary.
- **Least privilege at OS level**: Use `seccomp`, `AppArmor`, or `SELinux` profiles to sandbox each MCP server container. Block filesystem, network, and syscall access beyond what the tool actually needs.
- **No unauthenticated localhost exposure**: Even localhost-bound MCP servers require authentication. Do not assume same-host = trusted.
- **Origin header validation**: All HTTP-based MCP servers must validate the `Origin` header on every incoming connection. Reject requests from untrusted origins.
- **Pin versions**: Lock MCP server container image tags and package versions. Floating `latest` tags enable rug-pull-style supply chain attacks.
- **Audit tool definitions continuously**: Do not trust tool metadata only at deploy time. Monitor for changes to tool descriptions and alert on drift. Treat this like software supply chain security.
- **Secrets isolation**: MCP servers must never receive credentials broader than their specific function. Use Docker secrets with per-service scope, not shared env files.
- **Log everything, but safely**: Log all tool invocations and elevation events with correlation IDs. Never log authorization tokens, API keys, or session secrets.

#### For Developers (Building MCP Servers / Integrations)

- **Minimal initial scope**: Start with read-only, low-risk discovery operations. Require explicit authentication challenges for privileged operations.
- **OAuth2 with per-client consent**: Never use static OAuth client IDs with shared consent state. Each client must have its own consent record stored server-side.
- **Registry of approved clients**: MCP proxy servers must maintain a per-user registry of approved `client_id` values and check it before initiating any third-party auth flow.
- **Cryptographic signing**: Sign all MCP server artifacts. Build pipelines must include static analysis scanning before any release.
- **Tool description transparency**: Tool descriptions shown to the AI must also be visible to humans. Flag any divergence between user-visible and AI-visible instructions.
- **No dynamic tool redefinition**: Tools must not be able to rewrite their own metadata at runtime. Treat tool definitions as immutable after registration.

#### For Implementers (System Integrators / Architects)

- **Treat every MCP server as hostile until proven otherwise.** Apply defense-in-depth: gateway → scope → sandbox → log → review.
- **Single-server blast radius reduction**: Do not aggregate credentials for multiple services in a single MCP server. One server = one service domain.
- **Progressive trust model**: Grant capabilities incrementally. Revoke and re-challenge on scope elevation.
- **Defense in depth for localhost**: Even on loopback, require mutual authentication. Do not rely on network position as a trust signal.
- **Draw lessons from prior plugin ecosystems**: Apply the same scrutiny used for browser extensions, CI/CD plugins, and package registry packages to MCP servers.
- **Coordinate with standards bodies**: Track MCP specification evolution. Security properties are not stable; re-evaluate on each spec version change.

---

### Docker/Traefik-Specific Security Checklist for MCP Deployments

When adding any MCP server or AI automation service to this stack, verify:

- [ ] Container runs in a dedicated Docker network, not `t2_proxy` or `socket_proxy`
- [ ] `seccomp` or `AppArmor` profile applied in compose definition
- [ ] No host volume mounts beyond minimum required paths
- [ ] Image tag pinned to a specific digest or version (not `latest`)
- [ ] Secrets injected via Docker secrets, not environment variables
- [ ] Traefik middleware includes rate limiting and auth (Authelia or forward-auth)
- [ ] Origin header validation enabled on the MCP server
- [ ] No direct internet exposure of MCP management/admin endpoints
- [ ] Outbound network access restricted (no unrestricted egress)
- [ ] Logging configured to capture invocations but exclude token values

---

### Key References

- NSA CSI: [MCP Security Design Considerations](https://www.nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf) (May 2026)
- [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)
- [MCP Official Security Best Practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)
- CVE-2025-66414: DNS rebinding in MCP TypeScript SDK (CVSS 7.6)

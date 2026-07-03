# CLAUDE.md — docker-traefik Project Context

## Project Overview

This repository is a personal, multi-host **Docker Compose + Traefik reverse-proxy** homelab
stack (the public repo behind SmartHomeBeginner.com's Docker/Traefik guides). It is not an
application with a build/test pipeline — it's declarative infrastructure: Compose files, Traefik
dynamic config, and example scripts/configs that the owner syncs (via Syncthing) across 5 real
hosts and periodically pushes to GitHub.

There is **no CI, no test suite, and no package manager** in this repo. "Correctness" means: valid
YAML, consistent with the conventions below, and safe to `docker compose up` on the target host.

### The 5 hosts

| Prefix | Host | Hardware / OS |
|--------|------|---------------|
| `hs` | Home Server | Proxmox LXC, Ubuntu Server 22.04 |
| `mds` | Media/Database Server | Proxmox LXC, Ubuntu Server 22.04 |
| `ws` | Web Server (powers smarthomebeginner.com) | DigitalOcean VPS, Ubuntu Server 22.04 |
| `ds918` | Synology NAS | Synology DS918+ (DSM) |
| `dns` | DNS / AdBlock | Raspberry Pi 4B |

Each host has its own top-level entrypoint compose file (`docker-compose-$HOSTNAME.yml`) and its
own subdirectory of per-service Compose files under `compose/$HOSTNAME/`. The same service
(e.g. `sonarr.yml`) is frequently copy-pasted between hosts with only the host-scoped variables
differing — don't be surprised to find near-duplicate files under different host directories.

---

## Repository Layout

```
docker-compose-hs.yml          # Entrypoint compose file per host (networks, secrets, `include:` list)
docker-compose-mds.yml
docker-compose-ws.yml
docker-compose-ds918.yml
docker-compose-dns.yml

.env.example                   # Template for the real (gitignored) .env at repo root
CHANGELOG.md                   # Frozen/no-longer-maintained; see `git log` instead
README.md                      # Public-facing docs, app inventory, links to setup guides

compose/
  hs/  mds/  ws/  ds918/  dns/  # One Compose file per service, included by the host's entrypoint
  archives/                     # Retired/unmaintained service definitions, kept for reference

appdata/                       # Per-app persistent config — almost entirely gitignored
  traefik2/, traefik3/         #   Traefik dynamic config ("rules") lives here, keyed by host
    rules/$HOSTNAME/*.yml      #   Only tls-opts.yml, chain-*.yml, middlewares-*.yml are tracked
    rules/$HOSTNAME/*.yml.example  # App-specific router examples (real ones are gitignored)
  authelia/, nginx/, php/, picard/, rclone/   # Only *.example files are tracked here

archives/traefik_v1/           # Historical Traefik v1 / Swarm setups, unmaintained

custom/                        # Dockerfiles for images built locally (php7, csdash)

secrets_example/                # Template files for Docker secrets (real `secrets/` dir is gitignored)

scripts/                       # Example helper/systemd scripts, per host, tracked as *.example only

shared/config/
  bash_aliases                 # Actual, actively-synced bash aliases (tracked in full)
  bash_aliases.env.example     # Template for host-specific env vars sourced by bash_aliases
```

### The `.gitignore` allowlist model — READ THIS BEFORE ADDING FILES

This repo inverts the usual `.gitignore` pattern: **the root `.gitignore` starts with `*` and
`*/` (ignore everything), then explicitly un-ignores specific files and directories.** This is
intentional — it lets the owner sync real secrets, real Traefik rules, real `.env` files, and
real appdata across hosts via Syncthing in the *same* directories that are checked into git,
without ever committing the sensitive/live versions.

Practical consequences when editing this repo:

- Only `*.example` files are tracked inside `appdata/`, `secrets_example/`, `scripts/`, and
  `shared/config/` (with a few named exceptions like `shared/config/bash_aliases` and the
  Traefik `tls-opts.yml` / `chain-*.yml` / `middlewares-*.yml` files, which are tracked directly
  because they contain no secrets).
- If you add a new example/template file, you generally don't need to touch `.gitignore` — the
  `!*.example` patterns already un-ignore it. If you add a new top-level directory or a file that
  doesn't match an existing pattern, you **must** add an explicit `!` allowlist entry or it will
  silently never be committed.
- **Never commit real secrets, a real `.env`, or a populated `secrets/` directory.** If a `git
  status`/`git add` unexpectedly stages something under `appdata/`, `secrets/`, or `.env` (not
  `.env.example`), stop and check the `.gitignore` — that file should not be tracked.
- App-specific Traefik router examples live as `app-*.yml.example` in
  `appdata/traefik3/rules/$HOSTNAME/` — copy one, drop the `.example` suffix, and edit it locally
  to add a new app's routing (the real file stays gitignored).

---

## Compose Architecture

### Entrypoint files

Each `docker-compose-$HOSTNAME.yml` at the repo root defines, in order:

1. **`networks:`** — the shared Docker networks for that host (see Networking below).
2. **`secrets:`** — top-level Docker secrets, each pointing at `$DOCKERDIR/secrets/<name>`.
3. **`include:`** — the list of `compose/$HOSTNAME/*.yml` files that make up the stack. Services
   are grouped with `#` comment headers (CORE, SECURITY, FRONTEND, DOWNLOADERS, PVRS, MONITORING,
   ADMIN, UTILITIES, FILE MANAGEMENT, NETWORK, MAINTENANCE, etc.). Commented-out `#` lines are
   services that exist as files but aren't currently enabled on that host — this is the normal
   way to "disable" a service without deleting its Compose file.

To add a new service to a host: create `compose/$HOSTNAME/<service>.yml`, then add an `- compose/
$HOSTNAME/<service>.yml` line under the appropriate group comment in
`docker-compose-$HOSTNAME.yml`.

### Per-service Compose files

Each file under `compose/$HOSTNAME/` typically follows this shape (see `compose/hs/sonarr.yml`
as a representative example):

```yaml
services:
  <service>:
    image: ...
    container_name: <service>
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped   # or "no" for tools you start on demand
    profiles: ["<group>", "all"]
    networks:
      - t3_proxy               # or an explicit ipv4_address on the core proxy services
    volumes:
      - $DOCKERDIR/appdata/<service>:/config
      - "/etc/localtime:/etc/localtime:ro"
    environment:
      TZ: $TZ
      PUID: $PUID
      PGID: $PGID
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.<service>-rtr.entrypoints=websecure"
      - "traefik.http.routers.<service>-rtr.rule=Host(`<service>.$DOMAINNAME_HS`)"
      - "traefik.http.routers.<service>-rtr.middlewares=chain-oauth@file"
      - "traefik.http.routers.<service>-rtr.service=<service>-svc"
      - "traefik.http.services.<service>-svc.loadbalancer.server.port=<port>"
```

Conventions to follow:

- `container_name` matches the compose service key.
- `security_opt: [no-new-privileges:true]` on every service that runs as a long-lived daemon.
- **Profiles** gate which services start with a given `docker compose --profile <x> up`. Common
  profiles: `core` (Traefik, socket-proxy, auth, portainer — the stack's backbone), `all`
  (everything), `media`/`arrs` (the *arr apps + media tools), `downloads`, `apps`, `monitoring`,
  `dbs`. Every service belongs to `all` plus at least one more specific profile.
- Router naming: `<service>-rtr` for the router, `<service>-svc` for the service. Some apps
  expose a second `<service>-rtr-bypass` router (higher `priority`, matched via a header/query
  API key) to let mobile apps like LunaSea skip the OAuth/Authelia challenge — see
  `compose/hs/sonarr.yml` for the pattern.
- Middleware chains are referenced with the `@file` suffix (they're defined in the Traefik file
  provider, not as Docker labels) — see Traefik Rules below.

---

## Networking

Defined per-host in the entrypoint compose file. Typical set on `hs`/`mds`/`ws`:

- `socket_proxy` (`192.168.91.0/24`) — isolates the Docker socket; only `socket-proxy` and
  Traefik (and anything else that legitimately needs the Docker API) attach to it.
- `t3_proxy` (`192.168.90.0/24`) — the main reverse-proxy network. Traefik and socket-proxy get
  static `ipv4_address`es (`.254`); most application services just join `t3_proxy` with no static
  IP.
- `default` — plain bridge network for anything that doesn't need to be Traefik-routed.
- Older `t2_proxy` definitions are commented out — a leftover from the Traefik v2 setup, kept for
  reference when reviving `archives/traefik_v1`.

Docker API access is **never** given directly to application containers (no
`/var/run/docker.sock` bind mounts on app services). Everything that needs container introspection
(Traefik, Portainer, Watchtower-like tools) goes through `socket-proxy`
(`compose/$HOSTNAME/socket-proxy.yml`), which whitelists specific Docker API endpoints via env
vars (`CONTAINERS=1`, `EXEC=0`, `POST=1`, etc.). Follow the principle of least privilege when
adding a new consumer — enable only the endpoints that service actually needs, and never flip
`AUTH`/`SECRETS` on.

---

## Traefik Conventions

Two Traefik generations are present: `traefik3` (current — Traefik v3, referenced by the active
`compose/$HOSTNAME/traefik.yml` files) and `traefik2` (legacy, still present for hosts/services
not yet migrated). New work should target `traefik3` unless a specific host is still pinned to v2.

- **Static config** (entrypoints, providers, ACME/Let's Encrypt via Cloudflare DNS challenge,
  logging) lives as CLI `command:` args in `compose/$HOSTNAME/traefik.yml`.
- **Dynamic config** ("rules") lives in `appdata/traefik3/rules/$HOSTNAME/*.yml`, loaded via the
  file provider (`--providers.file.directory=/rules`, watched for changes). This is where
  middlewares and middleware **chains** are defined:
  - `middlewares-oauth.yml`, `middlewares-authelia.yml`, `middlewares-basic-auth.yml`,
    `middlewares-no-auth*.yml` — the actual auth middleware definitions.
  - `middlewares-rate-limit.yml`, `middlewares-secure-headers*.yml`,
    `middlewares-traefik-bouncer.yml`, `middlewares-buffering.yml` — cross-cutting middlewares.
  - `chain-oauth.yml`, `chain-authelia.yml`, `chain-basic-auth.yml`, `chain-no-auth*.yml` —
    named chains that compose the above into the single middleware a router references
    (`chain-oauth@file`, `chain-authelia@file`, etc). **When adding auth to a new router, pick an
    existing chain rather than inlining middlewares on the router's own labels.**
  - `tls-opts.yml` — shared TLS options referenced by the entrypoint's
    `tls.options=tls-opts@file`.
  - `app-*.yml.example` — one example per third-party/non-Docker app that needs a manually
    defined router (e.g. Proxmox VE, pfSense, a Synology package) instead of picking up
    Docker labels automatically.
- Certificates: DNS-01 challenge via Cloudflare (`CF_DNS_API_TOKEN_FILE`), wildcard + apex per
  domain, configured as indexed `tls.domains[N].main` / `.sans` args — see `compose/hs/traefik.yml`.
- Auth is chosen per-router, not globally: `chain-oauth` (Google OAuth via the `oauth`/`thomseddon
  traefik-forward-auth` service), `chain-authelia` (self-hosted Authelia), or `chain-no-auth`
  (public/no auth, used sparingly, e.g. `whoami`). Default to OAuth or Authelia for anything that
  isn't explicitly meant to be public.

---

## Secrets & Environment Variables

- **Docker secrets** (file-based, mounted at `/run/secrets/<name>`) are used for the truly
  sensitive values: Cloudflare API token, Authelia JWT/session/encryption keys, DB passwords,
  basic-auth htpasswd file, forward-auth config. Declared once at the top of each
  `docker-compose-$HOSTNAME.yml` under `secrets:`, each pointing at `$DOCKERDIR/secrets/<name>`
  (gitignored; `secrets_example/` shows the expected file format/content for each).
- **`.env`** (gitignored; `.env.example` is the tracked template) supplies everything else:
  ports, `PUID`/`PGID`/`TZ`, `DOCKERDIR`/`USERDIR`/`SECRETSDIR`, domain names, Cloudflare
  zone/email, per-app API keys, VNC passwords, notification tokens. `HOSTNAME` is expected to be
  set in the shell environment (used by `include:` and by `bash_aliases` to pick the right
  compose file/profile) — Synology requires `docker-compose` (hyphenated, older binary) instead
  of `docker compose` there too — see `bash_aliases`'s `case $HOSTNAME in ds918)`.
  - Note: some variables referenced by service files (e.g. `DATADIR`, `DOWNLOADSDIR`,
    `DOMAINNAME_HS`, `DOMAINNAME_1`) are **not** listed in `.env.example` even though they're
    required — `.env.example` is a partial/illustrative template, not an exhaustive schema. When
    adding a new required variable to a Compose file, add it to `.env.example` too so the template
    stays useful, but don't assume `.env.example` is currently complete.
- Never hardcode a credential, token, or domain into a tracked Compose file — reference an env var
  or a Docker secret instead, and add the corresponding placeholder to `.env.example` /
  `secrets_example/`.

---

## Day-to-Day Operations (bash_aliases)

`shared/config/bash_aliases` (tracked, actively synced across all 5 hosts via Syncthing) defines
the operational workflow. Key patterns, all wrapping `docker compose -f
docker-compose-$HOSTNAME.yml` (or `docker-compose` on `ds918`):

- `dcup` / `dcdown` — bring the full (`--profile all`) stack up/down.
- `dcrec <service>` — recreate one service or the whole stack.
- `dcstop` / `dcrestart` / `dcstart` / `dcpull` / `dclogs` — per-service or stack-wide lifecycle.
- `createcore` / `startcore` / `stopcore` — operate on just the `core` profile (Traefik, auth,
  socket-proxy, portainer — the minimum viable stack).
- Equivalent `*media`, `*downloads`, `*arrs`, `*dbs` alias sets for the other profiles.
- `cscli`/`cs*` aliases wrap CrowdSec's `cscli` for inspecting bans/decisions/alerts.
- `dp600` / `dp777` — lock down / open up permissions on `secrets/` and `.env` for editing.

When proposing operational changes (new service, new profile, changed port), keep this alias
model in mind — a new service should fit into an existing profile rather than requiring a brand
new alias family unless it's a genuinely new category of app.

---

## Conventions Summary (for any change in this repo)

1. **Scope changes to one host directory** (`compose/$HOSTNAME/`) unless the change is genuinely
   cross-host (e.g. a shared Traefik middleware chain pattern) — don't let an edit meant for `hs`
   leak into `mds`/`ws`/`ds918`/`dns` copies of the same service file.
2. **Never commit secrets, real `.env`, or real `appdata`.** Only `.example` files, and the
   explicitly-allowlisted tracked files, should ever be staged from those directories.
3. **Match existing label/router/service naming** (`<service>-rtr`, `<service>-svc`) and reuse
   existing middleware chains rather than inventing new ones per-service.
4. **Add every new service to a `profiles` list** (`all` plus one meaningful group) and wire it
   into the host's `include:` list in `docker-compose-$HOSTNAME.yml`.
5. **Update `.env.example` / `secrets_example/`** whenever a Compose file introduces a new
   required variable or secret, so the templates stay usable for other readers of this public
   repo.
6. **`archives/` and `compose/archives/`** are intentionally unmaintained — don't "fix" them
   unless specifically asked; they're kept only as a reference/starting point.
7. There is no build or test step to run. Validate YAML changes by checking indentation/structure
   carefully (e.g. `docker compose -f docker-compose-$HOSTNAME.yml config` would validate on a
   real host with a populated `.env`, but that's not runnable in this repo in isolation since
   `.env`/secrets aren't present here).

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

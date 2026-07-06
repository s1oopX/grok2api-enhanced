# grok2api Enhanced Deployment Variant

This document describes the deployment extensions included in this repository.
The upstream application remains the API gateway; this variant adds an
operator-oriented deployment layer for egress selection, private access, and
runtime network observability.

The implementation is intentionally separated from real deployment state. The
repository contains only reusable templates, Compose overlays, and sanitized
configuration examples.

## Distribution Model

This repository is a deployment overlay distribution. The default runtime model
is:

```text
upstream grok2api image
  + Compose overlays
  + admin static override
  + Mihomo dashboard assets
  + private access template
  + sanitized runtime examples
```

The optional application image workflow exists for maintainers who want to build
the full source tree from this repository. It is not required for the standard
overlay-based deployment path.

## Scope

This variant focuses on four operational concerns:

1. **Egress abstraction**: route application traffic through a stable internal
   proxy endpoint instead of binding runtime behavior to a single host network.
2. **Operator-controlled node selection**: delegate node-level selection to
   Mihomo and expose a dashboard for inspection and controlled switching.
3. **Private administrative surfaces**: place admin and dashboard routes behind
   an access gate rather than exposing them directly.
4. **Publication hygiene**: keep credentials, runtime data, account databases,
   node subscriptions, and deployment-specific hostnames out of version control.

## Component Inventory

| Component | File | Purpose |
| --- | --- | --- |
| Admin config override | `overrides/admin/config.html` | Adds fixed proxy choices for `http://privoxy:8118` and `http://mihomo:7890`. |
| Mihomo overlay | `docker-compose.mihomo.yml` | Adds the `mihomo` service and mounts the admin config override. |
| Mihomo example config | `mihomo/config.example.yaml` | Provides a placeholder-only Mihomo configuration with HTTP and SOCKS5 examples. |
| Private access overlay | `docker-compose.private.yml` | Adds an `access-gate` reverse proxy in front of the application. |
| Private access template | `nginx-private.example.conf` | Demonstrates allowlisted access to `/`, `/mihomo/`, and `/mihomo-api/`. |
| Egress notes | `docs/egress.md` | Documents routing modes and operational constraints. |
| Sanitization checklist | `docs/open-source-sanitization.md` | Lists files, patterns, and credentials that must not be published. |
| Architecture notes | `docs/architecture.md` | Defines the overlay distribution model and runtime boundaries. |

## Network Topology

```text
external client
  -> optional tunnel or reverse proxy
  -> access-gate
     -> grok2api:8000
     -> /mihomo/ static dashboard
     -> /mihomo-api/ -> mihomo:9090

grok2api
  -> selected internal proxy endpoint
     -> http://privoxy:8118
     -> http://mihomo:7890
```

`mihomo` exposes `7890` and `9090` only inside the Compose network. The
dashboard and controller API are intended to be reached through `access-gate`,
not through public host-port bindings.

## Egress Model

The application should treat outbound routing as a small set of stable internal
proxy endpoints.

| Mode | Proxy URL | Description |
| --- | --- | --- |
| Direct | empty / disabled | Use the host network path directly. |
| Privoxy path | `http://privoxy:8118` | Route through the Privoxy service and its configured upstream. |
| Mihomo path | `http://mihomo:7890` | Route through the node currently selected in Mihomo. |

For account-bound or session-bound workloads, keep the application and companion
network helpers on the same egress path. Switching exits during an active batch
can make runtime state difficult to reason about.

## Deployment Profiles

### Full Enhanced Profile

Use this profile when you want the application, private access gate, WARP/Privoxy
path, tunnel ingress, and Mihomo path available in one Compose project.

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf
# Place your local tunnel token at ./cloudflared.token when using the tunnel overlay.

docker compose \
  -f docker-compose.warp.yml \
  -f docker-compose.tunnel.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

### Private Mihomo Profile

Use this profile when you want the base application, tunnel ingress, private
access gate, and Mihomo egress path without the WARP/Privoxy services.

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf
# Place your local tunnel token at ./cloudflared.token when using the tunnel overlay.

docker compose \
  -f docker-compose.yml \
  -f docker-compose.tunnel.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

If you do not use the tunnel overlay, adapt `nginx-private.example.conf` to your
own ingress layer and remove the `cloudflared` service dependency from the
effective Compose configuration.

## Runtime Configuration

In the admin configuration page, select a fixed egress endpoint:

```text
egress strategy: single proxy
base proxy URL: http://privoxy:8118 or http://mihomo:7890
resource proxy URL: http://privoxy:8118 or http://mihomo:7890
```

Use the same proxy URL for base API traffic and resource downloads unless you
have a specific reason to separate those paths.

The Mihomo dashboard is served through:

```text
/mihomo/
```

The Mihomo controller is proxied internally through:

```text
/mihomo-api/
```

Replace `REPLACE_WITH_A_RANDOM_SECRET` in `mihomo/config.yaml` and
`REPLACE_WITH_MIHOMO_SECRET` in `nginx-private.conf` with the same strong secret.

## Operational Guidance

- Prefer one stable production egress at a time.
- Switch nodes deliberately between batches, not during active account refreshes
  or long-running requests.
- Keep `http://privoxy:8118` available as a simple fallback path when using
  Mihomo as the primary egress.
- Do not bind Mihomo controller ports directly to the public host interface.
- Treat `/admin/`, `/mihomo/`, and `/mihomo-api/` as administrative routes.
- Keep local runtime files outside git and regenerate them from examples on new
  hosts.

## Publication Safety

Do not commit:

- `.env`
- `data/`
- `logs/`
- `cloudflared.token`
- `nginx-private.conf`
- `mihomo/config.yaml`
- account databases
- proxy subscription URLs
- proxy node credentials
- provider tokens, cookies, API keys, or session material
- deployment-specific domains, IP addresses, or hostnames

Before publishing, run the checks in
`docs/open-source-sanitization.md` and review the staged file list manually.

## Attribution

This repository is based on
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api). The original
license and attribution are preserved. The added material in this variant is
focused on deployment composition, egress management, private access templates,
and publication-safe operational documentation.

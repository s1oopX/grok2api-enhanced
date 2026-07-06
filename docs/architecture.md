# Architecture

This repository is structured as a deployment overlay distribution for
grok2api. The upstream application image remains the core API gateway; this
repository adds a set of operational layers around it.

## Runtime Boundary

```text
external client
  -> ingress layer
  -> access-gate
     -> grok2api:8000
     -> /mihomo/ static dashboard
     -> /mihomo-api/ -> mihomo:9090

grok2api
  -> selected internal egress endpoint
     -> direct host network
     -> http://privoxy:8118
     -> http://mihomo:7890
```

The API gateway, egress selector, dashboard, and access controls are separate
concerns:

| Boundary | Owner | Notes |
| --- | --- | --- |
| API gateway | upstream grok2api image or optional local build | Handles API-compatible request processing. |
| Egress endpoint | application runtime config | Selects direct, Privoxy, or Mihomo as the outbound path. |
| Node selection | Mihomo | Owns proxy subscriptions, node health, node switching, and controller state. |
| Admin exposure | access-gate | Keeps admin and dashboard surfaces behind an allowlist or equivalent control. |
| Publication state | git repository | Contains examples and templates only, not runtime secrets or account data. |

## Compose Layers

| File | Layer | Responsibility |
| --- | --- | --- |
| `docker-compose.yml` | base | Starts the application with local data and log volumes. |
| `docker-compose.warp.yml` | egress | Adds WARP, Privoxy, companion network helper services, and proxy initialization. |
| `docker-compose.tunnel.yml` | ingress | Adds a tunnel client that depends on the application. |
| `docker-compose.private.yml` | access | Adds `access-gate` and routes ingress through the private reverse proxy. |
| `docker-compose.mihomo.yml` | egress and UI | Adds `mihomo`, mounts the admin config override, and exposes dashboard assets through `access-gate`. |

The full enhanced profile is:

```bash
docker compose \
  -f docker-compose.warp.yml \
  -f docker-compose.tunnel.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  config --services
```

The private Mihomo profile without WARP/Privoxy is:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.tunnel.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  config --services
```

## Image Strategy

The standard overlay path uses the upstream grok2api image and mounts the
enhanced admin configuration from `overrides/admin/config.html`.

The repository also keeps an optional application image workflow for maintainers
who want to build this source tree into a standalone image. That workflow is
scoped to application source, dependency, Dockerfile, and workflow changes. It is
not triggered by documentation-only or overlay-only changes.

## Egress Strategy

The application should target one stable internal proxy endpoint at a time:

| Endpoint | Meaning |
| --- | --- |
| empty / disabled | Direct host egress. |
| `http://privoxy:8118` | Privoxy path, with its configured upstream. |
| `http://mihomo:7890` | Mihomo mixed-port path, using the selected node. |

Mihomo owns node-specific configuration. The application should not store proxy
subscription URLs or individual node credentials.

## Access Strategy

Administrative routes should not be treated as public application endpoints:

- `/admin/`
- `/mihomo/`
- `/mihomo-api/`

`nginx-private.example.conf` demonstrates an allowlist gate using placeholder
documentation address ranges. Replace those placeholders with your own ingress
identity or network controls before use.

## Repository Hygiene

The repository should contain only reusable deployment artifacts. Runtime state
belongs outside git:

- local environment files
- account databases
- logs
- tunnel tokens
- private reverse-proxy config
- Mihomo runtime config
- proxy subscriptions and node credentials
- provider cookies, tokens, API keys, or session data

Use `docs/open-source-sanitization.md` before publishing changes.

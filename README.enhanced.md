# grok2api Enhanced Deployment Variant

This repository is an enhanced deployment-oriented variant of
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api).

The focus is operational maintainability for self-hosted deployments:

- visual egress switching between WARP/Privoxy and Mihomo
- a lightweight Mihomo node dashboard under `/mihomo/`
- a private access-gate example for admin surfaces
- conservative defaults for clearance refresh and account quota sync
- Docker Compose overlays that can be layered onto the base deployment

This variant does not include any deployment-specific hostnames, account data,
Cloudflare credentials, proxy subscription secrets, or runtime logs.

## Added Components

| Area | Change |
| --- | --- |
| Admin config | `proxy.egress.proxy_url` and `resource_proxy_url` expose fixed choices for `http://privoxy:8118` and `http://mihomo:7890`. |
| Mihomo UI | `/mihomo/` provides current node display, node switching, and delay testing. |
| Mihomo service | `docker-compose.mihomo.yml` adds a `mihomo` container exposing `7890` and `9090` only inside the Compose network. |
| Private access | `nginx-private.example.conf` shows a generic IP allowlist gate for admin and dashboard routes. |
| Examples | `mihomo/config.example.yaml` is a placeholder template; copy it to `mihomo/config.yaml` and fill in your own nodes. |

## Architecture

```text
Client
  -> private access gate
  -> grok2api
     -> FlareSolverr
     -> selected egress

Egress choices:
  http://privoxy:8118  -> WARP/Privoxy path
  http://mihomo:7890   -> Mihomo selected node
```

Keep FlareSolverr and grok2api on the same egress path. Mixing clearance from
one egress with API requests from another egress can cause session mismatch and
unstable upstream behavior.

## Quick Start

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf

docker compose \
  -f docker-compose.warp.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

Open the admin config page and set:

```text
Network proxy -> Egress proxy
mode: single_proxy
proxy_url: http://privoxy:8118 or http://mihomo:7890
resource_proxy_url: http://privoxy:8118 or http://mihomo:7890
```

Open the Mihomo dashboard:

```text
/mihomo/
```

Use one fixed node for production traffic. Avoid automatic random rotation for
account-bound workloads.

## Publication Safety

Do not commit:

- `.env`
- `data/`
- `logs/`
- `cloudflared.token`
- `nginx-private.conf`
- `mihomo/config.yaml`
- account databases
- proxy subscription URLs or node credentials

Run the checklist in `docs/open-source-sanitization.md` before publishing.

## Attribution

Original project:
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api)

License: MIT. Keep the original license and attribution when redistributing.

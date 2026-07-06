# grok2api-enhanced

An operations-focused deployment overlay distribution for self-hosted grok2api.

This repository does not replace the upstream API gateway. It packages a cleaner
deployment layer around it: configurable egress, a Mihomo dashboard,
private-access examples, and publication-safe sample configuration.

> For research, learning, and self-hosted deployment evaluation only. Follow the
> applicable service terms, platform rules, and local laws.

## Focus

This distribution is about deployment control rather than rewriting the upstream
application story. The standard path uses the upstream grok2api image plus
Compose overlays and runtime templates.

It focuses on:

- choosing a stable outbound path from the admin surface
- switching between Privoxy/WARP and Mihomo without editing runtime files
- exposing a Mihomo dashboard under a private route
- keeping admin and dashboard surfaces behind an access gate
- publishing example configuration without hostnames, accounts, tokens, logs, or
  proxy credentials

## Added Pieces

| Area | Addition |
| --- | --- |
| Admin config | Fixed egress choices for `http://privoxy:8118` and `http://mihomo:7890` |
| Mihomo dashboard | `/mihomo/` for current node, node switching, and latency checks |
| Compose overlay | Compose files for layering ingress, private access, and egress services around the base stack |
| Private access | `nginx-private.example.conf` as a generic allowlist gate example |
| Example config | `mihomo/config.example.yaml` with placeholders only |
| Publishing checklist | `docs/open-source-sanitization.md` for release hygiene |

## Architecture

```text
client
  -> optional tunnel or reverse proxy
  -> access-gate
     -> grok2api
     -> /mihomo/
     -> /mihomo-api/ -> mihomo:9090

grok2api
  -> selected internal proxy endpoint
     -> direct
     -> http://privoxy:8118
     -> http://mihomo:7890
```

For account-bound traffic, use one stable egress path in production. Random or
frequent node rotation can make session behavior harder to reason about.

## Quick Start

Create local runtime files from examples:

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf
# If the tunnel overlay is enabled, place your local tunnel token at ./cloudflared.token.
```

Start the full enhanced profile:

```bash
docker compose \
  -f docker-compose.warp.yml \
  -f docker-compose.tunnel.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

Start the private Mihomo profile without WARP/Privoxy:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.tunnel.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

## Egress Selection

In the admin config page, use:

```text
egress mode: single_proxy
proxy_url: http://privoxy:8118 or http://mihomo:7890
resource_proxy_url: http://privoxy:8118 or http://mihomo:7890
```

Typical choices:

| Option | Use when |
| --- | --- |
| `direct` | The server's native network path is reliable |
| `http://privoxy:8118` | You want the Privoxy/WARP path |
| `http://mihomo:7890` | You want the node selected in Mihomo |

The Mihomo dashboard is available at:

```text
/mihomo/
```

Place the admin surface and dashboard behind private access controls.

## Documents

- [Egress Strategy](egress.md)
- [Architecture](architecture.md)
- [Private Access](private-access.md)
- [Open Source Sanitization](open-source-sanitization.md)
- [Enhanced Variant Notes](../README.enhanced.md)

## Publication Safety

Do not commit:

- `.env`
- `data/`
- `logs/`
- `cloudflared.token`
- `nginx-private.conf`
- `mihomo/config.yaml`
- account databases
- real domains, IP addresses, or proxy nodes
- cookies, tokens, API keys, or subscription URLs

Run the checklist in `open-source-sanitization.md` before publishing.

## Upstream

This repository is based on
[jiujiu532/grok2api](https://github.com/jiujiu532/grok2api) and keeps the
original license and attribution.

New material in this repository is centered on deployment composition, egress
selection, private access examples, and sanitized public documentation.

## License

MIT. Keep the upstream license and attribution when redistributing.

# grok2api-enhanced

An operations-focused deployment variant for self-hosted grok2api.

This repository keeps the upstream API gateway behavior and adds a cleaner
deployment surface around it: configurable egress, a Mihomo dashboard,
private-access examples, and publication-safe sample configuration.

> For research, learning, and self-hosted deployment evaluation only. Follow the
> applicable service terms, platform rules, and local laws.

## Focus

This variant is about deployment control rather than rewriting the upstream
application story.

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
| Compose overlay | `docker-compose.mihomo.yml` for layering Mihomo onto the base stack |
| Private access | `nginx-private.example.conf` as a generic allowlist gate example |
| Example config | `mihomo/config.example.yaml` with placeholders only |
| Publishing checklist | `docs/open-source-sanitization.md` for release hygiene |

## Architecture

```text
client
  -> optional private access gate
  -> grok2api
     -> optional companion network helpers
     -> selected egress

egress choices
  direct
  http://privoxy:8118
  http://mihomo:7890
```

For account-bound traffic, use one stable egress path in production. Random or
frequent node rotation can make session behavior harder to reason about.

## Quick Start

Create local runtime files from examples:

```bash
cp .env.example .env
cp mihomo/config.example.yaml mihomo/config.yaml
cp nginx-private.example.conf nginx-private.conf
```

Start the enhanced stack:

```bash
docker compose \
  -f docker-compose.warp.yml \
  -f docker-compose.private.yml \
  -f docker-compose.mihomo.yml \
  up -d
```

Or layer only the Mihomo service onto the base stack:

```bash
docker compose \
  -f docker-compose.yml \
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

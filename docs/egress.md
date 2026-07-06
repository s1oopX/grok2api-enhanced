# Egress Management

This variant treats outbound networking as a first-class deployment concern.
The application keeps a single proxy entry point, while Mihomo owns node-level
selection.

## Recommended Modes

### WARP / Privoxy

```text
proxy_url = http://privoxy:8118
resource_proxy_url = http://privoxy:8118
```

Use this as a simple default egress path.

### Mihomo

```text
proxy_url = http://mihomo:7890
resource_proxy_url = http://mihomo:7890
```

Use this when you need a specific, operator-selected egress node.

## Why Not Put Nodes Directly Into grok2api?

The grok2api egress settings accept HTTP, HTTPS, and SOCKS5 proxy URLs.
Subscription nodes such as VLESS, Trojan, or Shadowsocks should be terminated by
Mihomo first. grok2api then talks to Mihomo through one stable HTTP proxy URL.

```text
grok2api and companion network helpers
  -> http://mihomo:7890
  -> selected Mihomo node
  -> upstream services
```

## Operational Rules

- Keep companion network helpers and API requests on the same egress.
- Use one fixed production node at a time.
- Avoid automatic random rotation for account-bound traffic.
- Switch nodes between batches, not during a running account refresh.
- Keep WARP/Privoxy available as a rollback path.

## Health Checks

From inside the Compose network:

```bash
docker exec grok2api python - <<'PY'
import requests
p = {"http": "http://mihomo:7890", "https": "http://mihomo:7890"}
for url in ["https://grok.com", "https://accounts.x.ai", "https://console.x.ai"]:
    r = requests.get(url, proxies=p, timeout=20, allow_redirects=True)
    print(url, r.status_code)
PY
```

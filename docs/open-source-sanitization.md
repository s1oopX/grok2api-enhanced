# Open Source Sanitization Checklist

Before publishing a fork or release artifact, verify that runtime-specific data
has not been included.

## Must Not Be Present

- `data/accounts.db`
- `data/config.toml`
- `.env`
- `cloudflared.token`
- `nginx-private.conf`
- `mihomo/config.yaml`
- `logs/`
- proxy subscription URLs
- account tokens
- real hostnames, public IPs, tunnel IDs, or provider-specific deployment names

## Suggested Scan

```bash
rg -n "cloudflared\\.token|accounts\\.db|app_key|api_key|Bearer [A-Za-z0-9_-]{20,}|sso=|vless://|trojan://|ss://|hysteria|tuic|REALITY|cf_clearance|__cf_bm"
```

False positives are expected for example names and source code identifiers.
Review every match before publishing.

## Safer Examples

Use RFC 5737 documentation IPs:

- `192.0.2.0/24`
- `198.51.100.0/24`
- `203.0.113.0/24`

Use placeholder hostnames:

- `example.com`
- `grok.example.com`

Use obvious placeholders:

- `REPLACE_ME`
- `YOUR_SECRET`
- `YOUR_PROXY_HOST`

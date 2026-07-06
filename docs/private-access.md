# Private Access Gate

`nginx-private.example.conf` demonstrates a small allowlist gate in front of
grok2api and the Mihomo dashboard.

The pattern is:

```text
Cloudflare Tunnel or reverse proxy
  -> nginx access gate
  -> grok2api
  -> /mihomo/ static dashboard
  -> /mihomo-api/ internal Mihomo controller proxy
```

## Setup

```bash
cp nginx-private.example.conf nginx-private.conf
```

Edit the allowlist:

```nginx
geo $http_cf_connecting_ip $private_allowed {
    default 0;
    203.0.113.10/32 1;
    2001:db8:1234::/64 1;
}
```

If your reverse proxy does not provide `CF-Connecting-IP`, replace
`$http_cf_connecting_ip` with the appropriate client IP variable for your
environment.

## Sensitive Routes

Treat these as administrative surfaces:

- `/admin/`
- `/mihomo/`
- `/mihomo-api/`

Do not expose them publicly without authentication and network access controls.

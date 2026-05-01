# Pi-hole Stack

This stack runs Pi-hole behind a Tailscale sidecar. It does not publish host
ports; DNS and the web UI are intended to be reached through the Tailscale node.

Default upstream DNS is Cloudflare:

```env
PIHOLE_DNS_UPSTREAMS=1.1.1.1;1.0.0.1
```

Start:

```sh
docker compose --env-file .env -f compose.yml up -d
```

Access the admin UI from a tailnet device:

```text
http://pihole/admin
```

Use the Tailscale IP or MagicDNS name as the DNS server for clients that should
use this Pi-hole instance.

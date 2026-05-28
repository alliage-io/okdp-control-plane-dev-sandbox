# DNS Configuration (Local)

To access the sandbox services via domain names (e.g. `*.okdp.dev-sandbox`), a local DNS server is deployed in the cluster on port **30053** (UDP).

This setup lets you skip editing `/etc/hosts` every time a new service is added.

## Resolver Configuration (macOS / Linux)

Create (or edit) `/etc/resolver/okdp.dev-sandbox` to route every `*.okdp.dev-sandbox` query to the local DNS server.

**File: `/etc/resolver/okdp.dev-sandbox`**

```text
nameserver 127.0.0.1
port 30053
```

*(Note: on macOS, make sure the `/etc/resolver` directory exists first: `sudo mkdir -p /etc/resolver`.)*

## Verification

Test resolution against any subdomain:

```bash
ping test.okdp.dev-sandbox
# Expected output:
# PING test.okdp.dev-sandbox (127.0.0.1): 56 data bytes
# 64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.059 ms
```

If resolution works, the services are reachable over HTTPS (e.g. `https://kubauth.okdp.dev-sandbox`).

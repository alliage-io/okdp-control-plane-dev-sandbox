# DNS Configuration (Local)

To access the sandbox services via domain names (e.g. `*.okdp.dev-sandbox`), a local DNS server is deployed in the cluster on port **30053** (UDP).

This setup lets you skip editing `/etc/hosts` every time a new service is added.

## Resolver Configuration (macOS)

Create (or edit) `/etc/resolver/okdp.dev-sandbox` to route every `*.okdp.dev-sandbox` query to the local DNS server.

**File: `/etc/resolver/okdp.dev-sandbox`**

```text
nameserver 127.0.0.1
port 30053
```

## Resolvectl Configuration (Linux — systemd-resolved)

Attach a routing domain to the Kind docker bridge so that only `*.okdp.dev-sandbox` queries are sent to the sandbox DNS server:

```bash
KIND_BRIDGE="br-$(docker network inspect kind -f '{{.Id}}' | cut -c1-12)"
sudo resolvectl dns "$KIND_BRIDGE" 127.0.0.1:30053
sudo resolvectl domain "$KIND_BRIDGE" '~okdp.dev-sandbox'
```

All other queries keep using your normal resolver (including VPN-provided DNS).

*   These settings do not survive a reboot — re-run the commands above after a restart.
*   To undo: `sudo resolvectl revert "$KIND_BRIDGE"`.
*   Persistent alternative: create `/etc/systemd/resolved.conf.d/okdp-dev-sandbox.conf` with:

    ```ini
    [Resolve]
    DNS=127.0.0.1:30053
    Domains=~okdp.dev-sandbox
    ```

    then `sudo systemctl restart systemd-resolved`. Prefer the per-link method above if your setup already defines **global** DNS servers (e.g. a VPN client), as global servers share a single scope and queries may be sent to the wrong one.

## Verification

Test resolution against any subdomain:

```bash
ping test.okdp.dev-sandbox
# Expected output:
# PING test.okdp.dev-sandbox (127.0.0.1): 56 data bytes
# 64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.059 ms
```

If resolution works, the services are reachable over HTTPS (e.g. `https://kubauth.okdp.dev-sandbox`).

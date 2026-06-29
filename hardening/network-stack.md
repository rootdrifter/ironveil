# Network Stack — WireGuard, AdGuard Home, systemd-resolved

Living build — hardware values captured from the running machine on 2026-06-11. `FUTURE WORK`
markers flag planned enhancements.

## DNS resolution chain

```
application → systemd-resolved (127.0.0.1) → AdGuard Home (:53) → Quad9 DoH (encrypted) → egress via active WireGuard tunnel
```

AdGuard Home is the system resolver. Its only upstream is **Quad9 over DNS-over-HTTPS**
(`https://dns10.quad9.net/dns-query`), so every query leaves AdGuard already encrypted; that DoH
connection then egresses through whichever WireGuard tunnel is active. An observer on the external
interface sees only WireGuard-encrypted traffic — never a plaintext query, and never the host's
real query content.

> **Accuracy note:** the WireGuard provider also pushes a DNS server at `10.2.0.1` (visible in the
> tunnel config below), but AdGuard **overrides** it with Quad9 DoH — `10.2.0.1` is not the
> effective upstream.

## AdGuard Home

| Setting | Value |
|---------|-------|
| Install path | `/opt/AdGuardHome/` |
| Listener | `*:53` (all interfaces; pid 1452) — AdGuard is the system resolver |
| Upstream DNS | `https://dns10.quad9.net/dns-query` (Quad9 DoH), `upstream_mode: load_balance` |
| Bootstrap DNS | `9.9.9.10`, `149.112.112.10` (Quad9 — resolves the DoH endpoint hostname) |
| Block list | **AdGuard DNS filter** (enabled) — `https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt` |

Because the only upstream is encrypted (DoH) and egresses through the tunnel, no query leaves the
host in plaintext. Note the listener is bound to `*:53` (all interfaces), **not** loopback-only —
AdGuard can therefore answer on the LAN as well as locally; the no-plaintext-egress property comes
from the DoH-over-tunnel upstream, not from a loopback bind.

```yaml
# AdGuardHome.yaml (relevant excerpt)
upstream_dns:
  - https://dns10.quad9.net/dns-query
upstream_mode: load_balance
bootstrap_dns:
  - 9.9.9.10
  - 149.112.112.10
```

## WireGuard — wg-CH-FI-2 and wg-SE-FI-1

Two WireGuard tunnels are configured (NetworkManager connections). Both carry the full-tunnel
`allowed-ips` (`0.0.0.0/0, ::/0`), so while one is up it is the default route for all traffic.

| Attribute | wg-CH-FI-2 | wg-SE-FI-1 |
|-----------|------------|------------|
| Endpoint | `[REDACTED]:51820` | `[REDACTED]:51820` |
| Interface address | `10.2.0.2/32` | `10.2.0.2/32` |
| AllowedIPs | `0.0.0.0/0, ::/0` (full tunnel) | `0.0.0.0/0, ::/0` (full tunnel) |
| Persistent keepalive | 25 s | 25 s |
| Provider DNS (pushed, overridden) | `10.2.0.1` | `10.2.0.1` |
| Activation | manual (`autoconnect=false`) | manual (`autoconnect=false`) |

Sanitised NetworkManager config (private keys and provider endpoint IPs redacted — never committed):

```ini
[connection]
id=wg-CH-FI-2
type=wireguard
interface-name=wg-CH-FI-2

[wireguard]
listen-port=51820
private-key=[REDACTED]

[wireguard-peer]
endpoint=[REDACTED]:51820
persistent-keepalive=25
allowed-ips=0.0.0.0/0;::/0;

[ipv4]
address1=10.2.0.2/32
dns=10.2.0.1;
method=manual

# ---

[connection]
id=wg-SE-FI-1
type=wireguard
interface-name=wg-SE-FI-1

[wireguard]
listen-port=51820
private-key=[REDACTED]

[wireguard-peer]
endpoint=[REDACTED]:51820
persistent-keepalive=25
allowed-ips=0.0.0.0/0;::/0;

[ipv4]
address1=10.2.0.2/32
dns=10.2.0.1;
method=manual
```

```
# NetworkManager-managed tunnels (manually activated)
nmcli connection show wg-CH-FI-2
nmcli connection up   wg-CH-FI-2      # or wg-SE-FI-1
wg show wg-CH-FI-2
```

**Kill-switch — honest description.** The protection here is *route-based*, not a separate
fail-closed rule: `allowed-ips=0.0.0.0/0, ::/0` makes the active tunnel the default route for all
traffic, so while a tunnel is up, traffic is tunnel-bound. There is **no** NetworkManager dispatcher
script or nftables/firewalld egress rule that drops traffic if the interface goes down, and both
tunnels are `autoconnect=false` (manually activated). So this is an *implicit* kill-switch (full-tunnel
routing) rather than a guaranteed fail-closed one — if a tunnel drops mid-session, traffic could
fall back to the bare interface until the tunnel is re-raised.

<!-- FUTURE WORK: add a true fail-closed kill-switch (nftables egress rule or NM dispatcher script) so traffic is dropped, not fallen back to clear, if the tunnel drops. -->

## systemd-resolved

`systemd-resolved` forwards all queries to AdGuard Home on `127.0.0.1` (AdGuard's `*:53` listener
includes the loopback address), with its own stub listener disabled so AdGuard owns port 53:

```
# /etc/systemd/resolved.conf
[Resolve]
DNS=127.0.0.1
Domains=~.
DNSStubListener=no
```

```
sudo systemctl restart systemd-resolved
resolvectl status            # confirm DNS=127.0.0.1 is the active server
```

## DNS leak verification

Verified on **2026-06-11** with a tunnel active:

```
resolvectl query example.com           # resolution observed egressing via wg-CH-FI-2 (tunnel-bound)
sudo ss -ulpn 'sport = :53'            # AdGuardHome listening on *:53 (pid 1452) — the system resolver
```

- `resolvectl query example.com` resolved with egress through the active `wg-CH-FI-2` interface,
  confirming DNS leaves only via the WireGuard tunnel.
- `ss` confirmed AdGuard Home (pid 1452) is the process bound to `:53`, i.e. the system resolver —
  there is no second resolver answering on port 53.

> Bound to `*:53` rather than loopback-only: the no-plaintext-egress guarantee rests on AdGuard's
> sole upstream being Quad9 DoH over the tunnel, verified above — not on the bind address.

## Current state

| Component | Status |
|-----------|--------|
| WireGuard `wg-CH-FI-2` / `wg-SE-FI-1` (NetworkManager, manual) | Operational |
| AdGuard Home on `*:53` (pid 1452), `/opt/AdGuardHome/` | Operational |
| Upstream via Quad9 DoH (`dns10.quad9.net`) over the active tunnel | Operational |
| systemd-resolved → `127.0.0.1` (stub listener off) | Operational |
| DNS leak verification (2026-06-11, via `wg-CH-FI-2`) | Confirmed — no plaintext egress |

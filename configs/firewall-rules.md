# Firewall Rules

Per-VLAN rule chains as configured in pfSense 2.8.1. Rules are evaluated **top-to-bottom** - first match wins.

---

## LAN - Native (10.10.1.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Block | TCP/UDP | LAN net | DOH_IPS alias | 443-853 | Block DoH / DoT |
| 2 | Block | TCP/UDP | LAN net | any | 53 | Block DNS to WAN |
| 3 | Pass | TCP | LAN net | MSFT_CONNECTIVITY alias | 443 | Microsoft connectivity check via WAN |
| 4 | Pass | any | LAN net | LAN net | any | Allow LAN-to-LAN (no VPN) |
| 5 | Pass | any | LAN net | any | any | Route via VPN_FAILOVER gateway |
| 6 | Block | IPv6 | LAN net | any | any | Drop all IPv6 |

> **Note:** LAN (VLAN 1) is the trunk native network used for switch management. It has no RFC1918 block - LAN devices can reach other VLANs by design. Rule #3 allows Microsoft connectivity checks to bypass VPN via WAN directly.

---

## VLAN 10 - Users (10.10.10.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN10 net | 10.10.10.1 | 53 | DNS to pfSense resolver |
| 2 | Pass | TCP | VLAN10 net | 10.10.10.1 | 443 | pfSense WebUI (HTTPS only) |
| 3 | Pass | ICMP | VLAN10 net | 10.10.10.1 | - | Gateway reachability (ping) |
| 4 | Pass | TCP | VLAN10 net | 10.10.20.5 | 9100 | Allow printing to printer |
| 5 | Block | TCP/UDP | VLAN10 net | DOH_IPS alias | 443-853 | Block DNS-over-HTTPS / DoT |
| 6 | Block | TCP/UDP | VLAN10 net | any | 53 | Block plain DNS to WAN |
| 7 | Block | any | VLAN10 net | RFC1918 alias | any | Block inter-VLAN traffic |
| 8 | Pass | any | VLAN10 net | any | any | Route via VPN_FAILOVER gateway |
| 9 | Block | IPv6 | any | any | any | Drop all IPv6 |

> **Note:** Gateway access is scoped to three specific services instead of blanket `any`: DNS (rule #1), WebUI (rule #2), and ICMP for ping (rule #3). SSH, API, and other pfSense services are no longer reachable from VLAN 10 - use VLAN 50 management for those. A temporary rule allowing VLAN 10 to reach switch management at 10.10.1.100 (TCP 80/443) also exists but is kept disabled - only enabled during active lab work.

---

## VLAN 20 - IoT (10.10.20.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN20 net | 10.10.20.1 | 53 | Allow DNS to pfSense only |
| 2 | Block | TCP/UDP | VLAN20 net | DOH_IPS alias | 443-853 | Block DoH / DoT |
| 3 | Block | TCP/UDP | VLAN20 net | any | 53 | Block plain DNS to WAN |
| 4 | Block | any | VLAN20 net | RFC1918 alias | any | Block inter-VLAN traffic |
| 5 | Pass | any | VLAN20 net | any | any | Route via VPN_FAILOVER gateway |
| 6 | Block | IPv6 | any | any | any | Drop all IPv6 |

---

## VLAN 30 - Guest (10.10.30.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN30 net | 10.10.30.1 | 53 | Allow DNS to pfSense only |
| 2 | Block | TCP/UDP | VLAN30 net | DOH_IPS alias | 443-853 | Block DoH / DoT |
| 3 | Block | TCP/UDP | VLAN30 net | any | 53 | Block plain DNS to WAN |
| 4 | Block | any | VLAN30 net | RFC1918 alias | any | Block inter-VLAN traffic |
| 5 | Pass | any | VLAN30 net | any | any | Route via VPN_FAILOVER gateway |
| 6 | Block | IPv6 | any | any | any | Drop all IPv6 |

---

## VLAN 40 - Lab (10.10.40.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN40 net | 10.10.40.1 | 53 | Allow DNS to pfSense only |
| 2 | Block | TCP/UDP | VLAN40 net | DOH_IPS alias | 443-853 | Block DoH / DoT |
| 3 | Block | TCP/UDP | VLAN40 net | any | 53 | Block plain DNS to WAN |
| 4 | Block | any | VLAN40 net | RFC1918 alias | any | Block inter-VLAN traffic |
| 5 | Pass | any | VLAN40 net | any | any | Route via VPN_FAILOVER gateway |
| 6 | Block | IPv6 | any | any | any | Drop all IPv6 |

> **Note:** VLAN 40 is excluded from Suricata IDS monitoring - Cisco lab protocols generate high false-positive noise.

---

## VLAN 50 - Management (10.10.50.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN50 net | 10.10.50.1 | 53 | DNS to pfSense resolver |
| 2 | Pass | TCP | VLAN50 net | 10.10.50.1 | 443 | pfSense WebUI (HTTPS only) |
| 3 | Pass | TCP | VLAN50 net | 10.10.50.1 | 22 | SSH to pfSense shell |
| 4 | Pass | ICMP | VLAN50 net | 10.10.50.1 | - | Gateway reachability (ping) |
| 5 | Block | TCP/UDP | VLAN50 net | any | 53 | Block plain DNS to WAN |
| 6 | Block | TCP/UDP | VLAN50 net | DOH_IPS alias | 443-853 | Block DoH / DoT |
| 7 | Pass | any | VLAN50 net | any | any | Direct internet - no VPN, full VLAN reach |
| 8 | Block | IPv6 | VLAN50 net | any | any | Drop all IPv6 |

> **Note:** VLAN 50 intentionally bypasses VPN and RFC1918 rules via rule #7 - direct WAN access and can reach all VLANs, for emergency admin access when something breaks. Gateway access (rules #1-#4) is scoped to the four services actually used: DNS, WebUI, SSH, and ping. Unlike VLAN 10, SSH is included here because VLAN 50 is the designated shell-access VLAN.

---

## Aliases Referenced

| Alias | Contents | Purpose |
|---|---|---|
| `DOH_IPS` | Known DoH provider IPs | Block encrypted DNS bypass on ports 443/853 |
| `RFC1918` | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | Block all inter-VLAN routing |

---

## Auto-Generated Rules

pfBlockerNG automatically creates floating permit rules allowing VLAN10_USERS, VLAN20_IOT, VLAN30_GUEST, and VLAN50_MGMT to reach the DNSBL VIP at 10.10.99.1 on ICMP and webserver ports 80 and 443. This lets blocked DNS lookups render the pfBlockerNG block page. VLAN40_LAB is excluded (isolated by design). WireGuard VPN interfaces and LAN are excluded (not client facing).

---

## Key Design Decisions

- **DNS is locked to pfSense** - clients cannot query external resolvers directly
- **DoH/DoT blocked** - prevents bypassing DNS controls via encrypted DNS
- **RFC1918 block enforces hard VLAN isolation** - no cross-VLAN traffic without an explicit rule above it
- **VPN_FAILOVER is the default gateway** - all passing traffic exits through Mullvad, not raw WAN
- **IPv6 fully blocked** - eliminates tunnel leak vectors
- **VLAN 50 is the only exception** - intentional, for emergency admin access with direct WAN and full internal reach

# VPN Failover

Dual Mullvad WireGuard tunnel configuration with automatic gateway failover in pfSense 2.8.1. Both exits are USA jurisdiction.

---

## Naming convention

Short, type-coded names. Format: `<TYPE>_<COUNTRY>_<TIER>`.

| Type prefix | pfSense object | UI location | Example |
|---|---|---|---|
| `TUN_` | WireGuard Tunnel | VPN > WireGuard > Tunnels | `TUN_USA_1` |
| `PEER_` | WireGuard Peer | VPN > WireGuard > Peers | `PEER_USA_1` |
| `INT_` | Assigned wg interface | Interfaces > Assignments | `INT_USA_1` |
| `GW_` | Gateway | System > Routing > Gateways | `GW_USA_1` |

Country = ISO 3-letter code (USA). Tier = 1 (active) or 2 (failover).

Older naming attempts (`GW_WG_MULLVAD_*`, `PEER_<COUNTRY>_<CITY>`) are deprecated. The current scheme separates the Tunnel object (`TUN_`) from the Peer sub-object (`PEER_`) and uses a tier suffix rather than a city code so exit locations are not exposed in public docs.

---

## Active tunnels (in failover group)

| Name | Provider | Tier | Role |
|---|---|---|---|
| `INT_USA_1` / `GW_USA_1` | Mullvad | 1 | Active, all traffic routes here by default |
| `INT_USA_2` / `GW_USA_2` | Mullvad | 2 | Standby, promoted automatically if Tier 1 fails |

Tier 1 was selected by latency test (Diagnostics > Ping from WAN to each candidate Mullvad endpoint). Tier 2 was selected for geographic and peering diversity from Tier 1.

---

## Gateway Group, VPN_FAILOVER

| Gateway | Tier | Behavior |
|---|---|---|
| `GW_USA_1` | 1 | Active, all traffic routes here by default |
| `GW_USA_2` | 2 | Standby, promoted automatically if Tier 1 fails |

Trigger Level: **Member Down**. pfSense monitors gateway health via ICMP. If `GW_USA_1` goes down, `GW_USA_2` is promoted with no manual intervention.

All internal firewall rules use `VPN_FAILOVER` as their gateway, not individual tunnel gateways. This ensures failover is automatic and transparent.

---

## Gateway monitoring

Each gateway needs a **unique** Monitor IP. pfSense installs one static route per Monitor IP pinning it to that gateway's interface. Two gateways sharing the same Monitor IP collide - the static route pins to only one of them, the other's monitor traffic exits the wrong tunnel, and that gateway falsely shows Offline.

### Verified working configuration (current, 2026-06-04)

| Gateway | Monitor IP | Notes |
|---|---|---|
| `GW_USA_1` | `100.64.0.31` | Mullvad in-tunnel DNS, family tier, distinct from `GW_USA_2`'s monitor |
| `GW_USA_2` | `100.64.0.32` | Mullvad in-tunnel DNS, family tier, distinct from `GW_USA_1`'s monitor |

The monitor packet stays inside the WireGuard tunnel end-to-end. It only completes successfully if Mullvad's internal network is functional, not just the tunnel handshake. This detects the case where the tunnel is up but Mullvad's internal services (DNS, routing) are dead, which the previous public-IP monitors did not catch. See [`dns-resilience.md`](dns-resilience.md) for the full failure mode and rationale.

Latency is higher than public-IP monitoring (~50-60ms vs ~10ms for `1.1.1.1`) and loss variance is slightly higher (0-3% vs 0%). Acceptable trade-off given the broader failure coverage.

### Previous configuration (pre 2026-06-04)

| Gateway | Monitor IP | Notes |
|---|---|---|
| `GW_USA_1` | `1.1.1.1` | Cloudflare DNS, distinct public IP |
| `GW_USA_2` | `9.9.9.9` | Quad9 DNS, distinct public IP |

ICMP travels through each respective tunnel (no ISP leak - your WAN sees only the encrypted WireGuard packets). The remote target sees Mullvad's exit IP, not yours. This was the verified path before the 2026-06-04 incident exposed its blind spot for tunnel-internal failures.

### Temporary monitor IPs during migration

When ADDING a new tunnel while old tunnels still use `1.1.1.1` and `9.9.9.9`, the new gateway's monitor IP must be different to avoid collision. Use `8.8.8.8` and `149.112.112.112` as temporary alternatives, then swap back to `1.1.1.1` / `9.9.9.9` after the old tunnels are deleted.

### Why distinct IPs per gateway (not the same IP for both)

`100.64.0.31` is Mullvad's per-server DNS IP, **reused across many servers**. If both gateways use the same IP as Monitor IP, pfSense pins the static route for that IP to whichever gateway was configured last - the other gateway's monitor traffic exits the wrong tunnel and marks the gateway falsely Offline. Verified in this lab on 2026-05.

The 2026-06-04 update sidesteps this by using **different per-gateway IPs** in Mullvad's set: `100.64.0.31` for `GW_USA_1`, `100.64.0.32` for `GW_USA_2`. Each gateway gets its own distinct route, no collision.

### Why NOT use `10.64.0.1` (Mullvad's tunnel gateway)

Same per-server reuse problem - it is also reused across servers. Distinct IPs per gateway is required.

---

## DNS Server Settings (System > General Setup)

| Setting | Value |
|---|---|
| DNS Servers | Two entries: `100.64.0.x` (from Tier 1 `.conf`) mapped to `GW_USA_1`, `100.64.0.x` (from Tier 2 `.conf`) mapped to `GW_USA_2` |
| DNS Server Override | Unchecked |
| DNS Resolution Behavior | Use local DNS (127.0.0.1), fall back to remote DNS Servers |
| Do not use DNS servers from DHCP | Checked |

Each tunnel gets its own DNS entry, mapped to its own gateway, so DNS follows the active VPN_FAILOVER tier without static-route collisions.

---

## DNS Configuration

DNS is locked to the VPN tunnels, it cannot leak to WAN under any condition, including during failover.

| DNS Server | Mapped Gateway | Effect |
|---|---|---|
| `100.64.0.x` (Tier 1) | `GW_USA_1` | Used while Tier 1 is active |
| `100.64.0.x` (Tier 2) | `GW_USA_2` | Used when failover promotes Tier 2 |

- DNS Resolver is bound strictly to internal interfaces and outbound only via the active VPN tunnel
- No fallback to ISP DNS exists

---

## How the Kill Switch Ties In

If all tunnels go down at the same time, the NAT and firewall layers make sure traffic is **blocked**, not leaked to WAN. See [`nat-rules.md`](nat-rules.md) for the full outbound NAT breakdown.

---

## WireGuard config explained (plain English)

Every Mullvad WireGuard config has the same structure:

```ini
[Interface]
# Device: <Mullvad-assigned label>
PrivateKey = <REDACTED, lives in password manager>
Address = 10.x.3.xxx/32
DNS = 100.64.0.x

[Peer]
PublicKey = <mullvad-server-public-key>
AllowedIPs = 0.0.0.0/0
Endpoint = x.x.x.x:51820
```

### `[Interface]` block - YOUR side of the tunnel

| Line | What it is | Where it comes from |
|---|---|---|
| `# Device: <label>` | Friendly device label assigned by Mullvad when you generated the keypair on their site. Comment only, ignored by WireGuard | Mullvad account, "WireGuard configuration" page |
| `PrivateKey` | YOUR secret. Mathematically paired with a public key Mullvad has on file. NEVER commit or share | Generated by Mullvad (or by you locally) when you created the device entry |
| `Address` | The IP YOUR side of the tunnel uses inside the tunnel. The `/32` means it is a single host address, not a subnet. This is private to you, only meaningful inside the encrypted tunnel | Assigned by Mullvad |
| `DNS` | The DNS server you will use while this tunnel is up. `100.64.0.x` is Mullvad's internal DNS reachable only through the tunnel | Mullvad standard |

### `[Peer]` block - MULLVAD'S side of the tunnel

| Line | What it is | Where it comes from |
|---|---|---|
| `PublicKey` | Mullvad's server's public key. Your client uses this to encrypt outgoing traffic so only that specific server can decrypt | Public, listed on Mullvad's server page |
| `AllowedIPs = 0.0.0.0/0` | "Send ALL my traffic through this tunnel" (full-tunnel VPN). If this were `10.0.0.0/8`, only that range would route through the tunnel (split-tunnel) | Standard for full-tunnel VPN |
| `Endpoint` | The actual public IP and UDP port of the Mullvad server. Your client connects here to establish the encrypted tunnel | Public, listed on Mullvad's server page |

### How to read a Mullvad config in one line

> "I am `Address`, my private key proves it. Send all my traffic (`AllowedIPs`) to the server at `Endpoint`, which proves it is itself with `PublicKey`. Resolve names through `DNS`."

That is the entire WireGuard handshake summarized in plain English.

---

## Adding a new Mullvad tunnel (procedure)

Use the naming convention from the top of this doc. The numeric tier is assigned in step 8.

### 1. VPN > WireGuard > Tunnels > Add Tunnel

- Enable Tunnel: check
- Description: `TUN_USA_<N>` (next free tier number)
- Listen Port: next free port unique across all tunnels on this box (e.g. 51822, 51823)
- Private Key: paste `[Interface] PrivateKey` from the `.conf` file
- Interface Addresses > Add Address:
  - Address: `.conf` `Address` without `/32`
  - CIDR: `32`
  - Description: `INT_USA_<N>`
- Save Tunnel

### 2. VPN > WireGuard > Peers > Add Peer

- Enable Peer: check
- Tunnel: select `TUN_USA_<N>` (the tunnel from step 1)
- Description: `PEER_USA_<N>`
- Dynamic Endpoint: **UNCHECK** (Mullvad uses static endpoints)
- Endpoint: IP portion of `.conf` `Endpoint`
- Port: port portion of `.conf` `Endpoint` (typically 51820)
- Keep Alive: `25` (standard WireGuard recommendation, keeps NAT alive)
- Public Key: paste `[Peer] PublicKey` from `.conf`
- Pre-shared Key: blank (Mullvad does not use PSK)
- Allowed IPs > Add Allowed IP:
  - Subnet: `0.0.0.0`
  - CIDR: `0`
  - Description: `fulltunnel`
- Save Peer

### 3. Interfaces > Assignments

- Add the new wg interface with Description `INT_USA_<N>`
- Open the interface > Enable, IPv4 Static, paste `.conf` `Address` (with `/32`)
- MTU: `1420` (WireGuard standard, 1500 - 80 bytes overhead)
- Block private networks: UNCHECK (Mullvad DNS uses CGNAT, tunnel uses RFC1918)
- Block bogon networks: UNCHECK

### 4. System > Routing > Gateways > Add Gateway

- Interface: `INT_USA_<N>`
- Name: `GW_USA_<N>`
- Gateway: `.conf` `Address` without `/32`
- Monitor IP: a distinct public DNS IP not used by any other gateway. Do NOT use Mullvad's `100.64.0.x`, it is reused across servers and breaks pfSense static routing

### 5. System > General Setup > DNS Servers

- Add the new `100.64.0.x` DNS from the `.conf`, map it to `GW_USA_<N>`
- Tick "Do not use DNS servers from DHCP"

### 6. Firewall > NAT > Outbound

- Copy each existing tunnel's 6 outbound NAT rules
- On each copy, change Interface to `INT_USA_<N>`, NAT Address to `INT_USA_<N> address`, Description to match
- Apply Changes

### 7. System > Routing > Gateway Groups > Edit `VPN_FAILOVER`

- Add `GW_USA_<N>` at the desired tier

### 8. Test

- Traceroute from a client, confirm the exit IP via [ipleak.net](https://ipleak.net) and [Mullvad Check](https://mullvad.net/en/check)
- Force failover by disabling the active tunnel briefly, confirm traffic flips to the new tier

---

## Common handshake breakers

If a tunnel fails to come up, check Status > WireGuard > Peers for "Latest handshake" age. If "Never" or stale, the tunnel is not actually up. Most common causes (in order):

1. Listen Port collision (two tunnels using the same port)
2. Private Key paste error (extra whitespace, missing character)
3. Public Key paste error
4. Dynamic Endpoint left CHECKED (no endpoint to send to)
5. Endpoint IP or Port wrong
6. Outbound UDP blocked by an upstream firewall
7. Local clock badly skewed (cryptographic timestamps reject)
8. Pre-shared Key mismatch (one side has it, other does not)

---

## Verified Behavior

- Failover Tier 1 > Tier 2 confirmed under simulated tunnel failure (manual peer disable)
- Zero DNS/IP/WebRTC leaks confirmed via [ipleak.net](https://ipleak.net) and [Mullvad Check](https://mullvad.net/en/check) on both tunnels
- DNS remains on Mullvad servers throughout failover, no ISP DNS exposure
- Each gateway health-checks via a distinct public DNS IP (`GW_USA_1` = 1.1.1.1, `GW_USA_2` = 9.9.9.9). ICMP routes through respective tunnel, no ISP leak

---

## Security notes

- **Never commit `.conf` files or the `PrivateKey`** to git. The `.conf` extension is already in `.gitignore`
- Public keys and endpoints ARE safe to commit (they are public information by design)
- The `Address` field (e.g. `10.x.3.xxx/32`) is semi-sensitive, redact it in docs
- **Exit city, server hostname, and provider account number are PRIVATE** and should never appear in this repo. Use tier numbers (`USA_1`, `USA_2`) only.
- If a private key leaks, regenerate the device entry in Mullvad and rotate

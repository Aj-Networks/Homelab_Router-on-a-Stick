# VPN Failover

Dual Mullvad WireGuard tunnel configuration with automatic gateway failover in pfSense 2.8.1. Both exits are EU jurisdiction by design.

---

## Naming convention

Short, location-coded names. Format: `<TYPE>_<COUNTRY>_<CITY>`.

| Type prefix | Used for | Example |
|---|---|---|
| `PEER_` | WireGuard peer entry in pfSense | `PEER_SWE_MMA` |
| `INT_` | Assigned wg interface | `INT_SWE_MMA` |
| `GW_` | Gateway in System > Routing > Gateways | `GW_SWE_MMA` |

Country = ISO 3-letter code (SWE, DEU, NLD, CHE, USA, etc.). City = airport code (MMA, FRA, AMS, ZRH, NYC, etc.).

Old long names (`GW_WG_MULLVAD_SWE_MMA`, etc.) are deprecated. Rename anything that still uses the old format on next touch.

---

## Active tunnels (in failover group)

| Name | Provider | Exit City | Tier | Role |
|---|---|---|---|---|
| `INT_SWE_MMA` / `GW_SWE_MMA` | Mullvad | Malmö, Sweden | 1 | Primary, all traffic routes here by default |
| `INT_DEU_FRA` / `GW_DEU_FRA` | Mullvad | Frankfurt, Germany | 2 | Failover, promoted automatically if Tier 1 fails |

Retired: `VPN_CHI` (deleted), `VPN_NYC` (disabled, kept as break-glass US fallback config).

---

## Gateway Group, VPN_FAILOVER

| Gateway | Tier | Behavior |
|---|---|---|
| `GW_SWE_MMA` | 1 | Active, all traffic routes here by default |
| `GW_DEU_FRA` | 2 | Standby, promoted automatically if Tier 1 fails |

Trigger Level: **Member Down**. pfSense monitors gateway health via ICMP. If `GW_SWE_MMA` goes down, `GW_DEU_FRA` is promoted with no manual intervention.

All internal firewall rules use `VPN_FAILOVER` as their gateway, not individual tunnel gateways. This ensures failover is automatic and transparent.

---

## Gateway monitoring

Each gateway needs a **unique** Monitor IP. pfSense installs one static route per Monitor IP pinning it to that gateway's interface. Two gateways sharing the same Monitor IP collide - the static route pins to only one of them, the other's monitor traffic exits the wrong tunnel, and that gateway falsely shows Offline.

### Verified working configuration

| Gateway | Monitor IP | Notes |
|---|---|---|
| `GW_SWE_MMA` | `1.1.1.1` | Cloudflare DNS, distinct public IP |
| `GW_DEU_FRA` | `9.9.9.9` | Quad9 DNS, distinct public IP |

ICMP travels through each respective tunnel (no ISP leak - your WAN sees only the encrypted WireGuard packets). The remote target sees Mullvad's exit IP, not yours.

### Why NOT use Mullvad's `100.64.0.31` as monitor

`100.64.0.31` is Mullvad's per-server DNS IP, **reused across many servers**. If both gateways use it as Monitor IP, pfSense pins the static route for `100.64.0.31` to whichever gateway was configured last - the other gateway's monitor traffic exits the wrong tunnel and marks the gateway falsely Offline. Verified in this lab on 2026-05.

### Why NOT use `10.64.0.1` (Mullvad's tunnel gateway)

Same issue - it is also reused across servers. Distinct IP per gateway is required.

### If you want fully in-tunnel monitoring later

You would need a unique IP per Mullvad server that responds to ICMP. Mullvad does not publish such addresses. Public DNS (above) is the pragmatic choice and is the verified path.

---

## DNS Server Settings (System > General Setup)

| Setting | Value |
|---|---|
| DNS Servers | Single entry: `100.64.0.31`, Gateway = **default** (blank) |
| DNS Server Override | Unchecked |
| DNS Resolution Behavior | Use local DNS (127.0.0.1), fall back to remote DNS Servers |

**Why a single entry with default gateway:** Mullvad reuses `100.64.0.31` across servers. A second entry with the same IP but a different gateway forces pfSense to pin a static route to only one gateway, which breaks the other gateway's DNS path during failover. Leaving Gateway = default lets DNS follow the `VPN_FAILOVER` group's active tier transparently.

---

## DNS Configuration

DNS is locked to the VPN tunnels, it cannot leak to WAN under any condition, including during failover.

| DNS Server | Mapped Gateway | Effect |
|---|---|---|
| `100.64.0.31` | default (blank) | Follows the active `VPN_FAILOVER` tier - MMA primary, FRA on failover |

- DNS Resolver is bound strictly to internal interfaces and outbound only via the active VPN tunnel
- System DNS uses one Mullvad address with default gateway (not pinned to a specific tunnel) so DNS fails over with the gateway group
- No fallback to ISP DNS exists

---

## How the Kill Switch Ties In

If all tunnels go down at the same time, the NAT and firewall layers make sure traffic is **blocked**, not leaked to WAN. See [`nat-rules.md`](nat-rules.md) for the full outbound NAT breakdown.

---

## WireGuard config explained (plain English)

Every Mullvad WireGuard config has the same structure. Using the SWE_MMA file as a worked example:

```ini
[Interface]
# Device: Superb Sloth
PrivateKey = <REDACTED, lives in password manager>
Address = 10.x.3.xxx/32
DNS = 100.64.0.31

[Peer]
PublicKey = vi0PPk0ZCDvDMCSQD0mctmPFFH7NiawLxJquyPIGwAY=
AllowedIPs = 0.0.0.0/0
Endpoint = 45.x.59.xx:51820
```

### `[Interface]` block - YOUR side of the tunnel

| Line | What it is | Where it comes from |
|---|---|---|
| `# Device: Superb Sloth` | Friendly device label assigned by Mullvad when you generated the keypair on their site. Comment only, ignored by WireGuard | Mullvad account, "WireGuard configuration" page |
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

Use the naming convention from the top of this doc. Example uses Frankfurt.

1. Mullvad account > WireGuard configuration > pick city > Download config (`.conf` file)
2. pfSense > VPN > WireGuard > Tunnels > Add Tunnel named `PEER_DEU_FRA`
   - Listen Port: leave blank (random)
   - Interface Keys > Private Key: paste from the `.conf` file `PrivateKey`
3. Add Peer to the tunnel
   - Public Key: paste from `.conf` file `[Peer] PublicKey`
   - Endpoint: paste `Endpoint` value (IP + port)
   - Allowed IPs: `0.0.0.0/0` (full-tunnel)
4. Interfaces > Assignments > Add the new wg interface, description `INT_DEU_FRA`
5. Interfaces > the new interface > Enable, set IPv4 = Static, fill `Address` value from `.conf` (e.g. `10.x.3.xxx/32`)
6. System > Routing > Gateways > Add `GW_DEU_FRA`
   - Gateway IP: the `.conf` `Address` minus `/32`
   - Monitor IP: a distinct public DNS IP not used by any other gateway (e.g. `9.9.9.9` if `1.1.1.1` is taken). Do NOT use Mullvad's `100.64.0.x` here, it is reused across servers and breaks pfSense static routing
7. System > General Setup > DNS Servers > add the new `100.64.0.x` DNS from the `.conf`, map it to the new gateway, check "Do not use DNS servers from DHCP"
8. Firewall > NAT > Outbound > add hybrid/manual NAT rule mapping internal nets to the new wg interface
9. System > Routing > Gateway Groups > Edit `VPN_FAILOVER` > add the new gateway at the desired tier
10. Test: traceroute from a client, confirm the exit IP via [ipleak.net](https://ipleak.net) and [Mullvad Check](https://mullvad.net/en/check)

---

## Verified Behavior

- Failover SWE_MMA > DEU_FRA confirmed under simulated tunnel failure
- Zero DNS/IP/WebRTC leaks confirmed via [ipleak.net](https://ipleak.net) and [Mullvad Check](https://mullvad.net/en/check) on both tunnels
- DNS remains on Mullvad servers throughout failover, no ISP DNS exposure
- Each gateway health-checks via a distinct public DNS IP (MMA = 1.1.1.1, FRA = 9.9.9.9). ICMP routes through respective tunnel, no ISP leak

---

## Security notes

- **Never commit `.conf` files or the `PrivateKey`** to git. The `.conf` extension is already in `.gitignore`
- Public keys and endpoints ARE safe to commit (they are public information by design)
- The `Address` field (e.g. `10.x.3.xxx/32`) is semi-sensitive, redact it in docs
- If a private key leaks, regenerate the device entry in Mullvad and rotate

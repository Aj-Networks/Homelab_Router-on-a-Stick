# VPN Failover

Dual Mullvad WireGuard tunnel configuration with automatic gateway failover in pfSense 2.8.1.

---

## Tunnels

| Name | Provider | Exit City | WireGuard Port | Role |
|---|---|---|---|---|
| VPN_CHI | Mullvad | Chicago, IL | 51820 | Primary - Tier 1 |
| VPN_NYC | Mullvad | New York City, NY | 51821 | Failover - Tier 2 |

---

## Gateway Group - VPN_FAILOVER

| Gateway | Tier | Behavior |
|---|---|---|
| GW_VPN_CHI | 1 | Active - all traffic routes here by default |
| GW_VPN_NYC | 2 | Standby - promoted automatically if Tier 1 fails |

Trigger: pfSense monitors gateway health via ICMP. If `GW_VPN_CHI` goes down, `GW_VPN_NYC` is promoted to active with no manual intervention.

All internal firewall rules use `VPN_FAILOVER` as their gateway - not individual tunnel gateways. This ensures failover is automatic and transparent.

---

## DNS Configuration

DNS is locked to the VPN tunnels - it cannot leak to WAN under any condition, including during failover.

| DNS Server | Mapped Interface | Exit |
|---|---|---|
| 100.64.0.1 | VPN_CHI | Mullvad Chicago |
| 100.64.0.2 | VPN_NYC | Mullvad NYC |

- DNS Resolver is bound strictly to internal interfaces and outbound only via VPN_CHI and VPN_NYC
- System DNS uses the two Mullvad addresses above
- No fallback to ISP DNS exists

---

## How the Kill Switch Ties In

If both tunnels go down at the same time, the NAT and firewall layers make sure traffic is **blocked** - not leaked to WAN. See [`nat-rules.md`](nat-rules.md) for the full outbound NAT breakdown.

---

## Verified Behavior

- Failover CHI > NYC confirmed under simulated tunnel failure
- Zero DNS/IP/WebRTC leaks confirmed via [ipleak.net](https://ipleak.net) and [Mullvad Check](https://mullvad.net/en/check) ([ipleak screenshot](../screenshots/ipleak.png)) ([mullvad screenshot](../screenshots/mullvad.png)) on both tunnels
- DNS remains on Mullvad servers throughout failover - no ISP DNS exposure

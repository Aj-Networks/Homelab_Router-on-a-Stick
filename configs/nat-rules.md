# NAT Rules

All 12 manual Outbound NAT rules as configured in pfSense 2.8.1. Mode is set to **Manual Outbound NAT**.

There are **zero rules mapping internal subnets to WAN** - this is the foundation of the kill switch.

---

## Outbound NAT Rules

### VPN_CHI (Chicago - Primary Tier 1)

| # | Interface | Source | Translation | Notes |
|---|---|---|---|---|
| 1 | VPN_CHI | 10.10.1.0/24 | VPN_CHI address | LAN Native > Chicago |
| 2 | VPN_CHI | 10.10.10.0/24 | VPN_CHI address | VLAN10 Users > Chicago |
| 3 | VPN_CHI | 10.10.20.0/24 | VPN_CHI address | VLAN20 IoT > Chicago |
| 4 | VPN_CHI | 10.10.30.0/24 | VPN_CHI address | VLAN30 Guest > Chicago |
| 5 | VPN_CHI | 10.10.40.0/24 | VPN_CHI address | VLAN40 Lab > Chicago |
| 6 | VPN_CHI | 10.10.50.0/24 | VPN_CHI address | VLAN50 Management > Chicago |

### VPN_NYC (New York City - Failover Tier 2)

| # | Interface | Source | Translation | Notes |
|---|---|---|---|---|
| 7 | VPN_NYC | 10.10.1.0/24 | VPN_NYC address | LAN Native > NYC |
| 8 | VPN_NYC | 10.10.10.0/24 | VPN_NYC address | VLAN10 Users > NYC |
| 9 | VPN_NYC | 10.10.20.0/24 | VPN_NYC address | VLAN20 IoT > NYC |
| 10 | VPN_NYC | 10.10.30.0/24 | VPN_NYC address | VLAN30 Guest > NYC |
| 11 | VPN_NYC | 10.10.40.0/24 | VPN_NYC address | VLAN40 Lab > NYC |
| 12 | VPN_NYC | 10.10.50.0/24 | VPN_NYC address | VLAN50 Management > NYC |

---

## Key Design Decisions

- **Manual mode is required** - Auto Outbound NAT would create WAN rules automatically, breaking the kill switch
- **Every subnet has a rule on both tunnels** - ensures clean failover with no gap
- **No WAN NAT rules exist** - if both VPN tunnels drop, traffic is blocked, not leaked
- **VLAN 50 NAT rules (6 and 12) are intentionally unused** - VLAN 50 routes through WAN directly by firewall design, so these rules never match. They're only there to keep the subnet list consistent across both tunnels.

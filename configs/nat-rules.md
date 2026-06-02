# NAT Rules

All 12 manual Outbound NAT rules as configured in pfSense 2.8.1. Mode is set to **Manual Outbound NAT**.

There are **zero explicit WAN outbound NAT rules**, this is the foundation of the kill switch. VLAN 50 reaches WAN via the system default gateway without a manual NAT rule.

---

## Outbound NAT Rules

### INT_USA_1 (Tier 1, Active)

| # | Interface | Source | Translation | Notes |
|---|---|---|---|---|
| 1 | INT_USA_1 | 10.10.1.0/24 | INT_USA_1 address | LAN Native > VPN_1 |
| 2 | INT_USA_1 | 10.10.10.0/24 | INT_USA_1 address | VLAN10 Users > VPN_1 |
| 3 | INT_USA_1 | 10.10.20.0/24 | INT_USA_1 address | VLAN20 IoT > VPN_1 |
| 4 | INT_USA_1 | 10.10.30.0/24 | INT_USA_1 address | VLAN30 Guest > VPN_1 |
| 5 | INT_USA_1 | 10.10.40.0/24 | INT_USA_1 address | VLAN40 Lab > VPN_1 |
| 6 | INT_USA_1 | 10.10.50.0/24 | INT_USA_1 address | VLAN50 Management > VPN_1 |

### INT_USA_2 (Tier 2, Failover)

| # | Interface | Source | Translation | Notes |
|---|---|---|---|---|
| 7 | INT_USA_2 | 10.10.1.0/24 | INT_USA_2 address | LAN Native > VPN_2 |
| 8 | INT_USA_2 | 10.10.10.0/24 | INT_USA_2 address | VLAN10 Users > VPN_2 |
| 9 | INT_USA_2 | 10.10.20.0/24 | INT_USA_2 address | VLAN20 IoT > VPN_2 |
| 10 | INT_USA_2 | 10.10.30.0/24 | INT_USA_2 address | VLAN30 Guest > VPN_2 |
| 11 | INT_USA_2 | 10.10.40.0/24 | INT_USA_2 address | VLAN40 Lab > VPN_2 |
| 12 | INT_USA_2 | 10.10.50.0/24 | INT_USA_2 address | VLAN50 Management > VPN_2 |

### WAN (VLAN 50 escape hatch)

| # | Interface | Source | Translation | Notes |
|---|---|---|---|---|
| 13 | WAN | 10.10.50.0/24 | WAN address | VLAN50_MGMT > WAN direct (admin escape hatch) |

---

## Key Design Decisions

- **Manual mode is required**, Auto Outbound NAT would create WAN rules automatically, breaking the kill switch
- **Every subnet has a rule on both tunnels**, ensures clean failover with no gap
- **No WAN NAT rules exist for client VLANs**, if both VPN tunnels drop, traffic is blocked, not leaked
- **VLAN 50 NAT rules (6 and 12) are intentionally unused for tunnel egress**, VLAN 50 routes through WAN directly by firewall design via rule 13, so the tunnel rules never match. They're kept to keep the subnet list symmetric across both tunnels.

---

## Adding rules for a new tunnel

When adding a third or replacement tunnel, copy the existing 6 rules from one tier, change Interface and NAT Address to the new `INT_USA_<N>`, update the Description suffix. Repeat for every VLAN subnet. Outbound NAT does NOT support gateway groups, so each VPN interface needs its own complete set of 6 rules.

# VLAN Assignments

Interface, subnet, and purpose map for the Router-on-a-Stick topology. All VLANs ride a single 802.1Q trunk on `igb1` (Port 7). Subnet third-octet always matches the VLAN ID.

---

## Interface Map

| Interface | VLAN | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| igb0 | n/a | DHCP/ISP | n/a | WAN |
| igb1 | 1 (Native) | 10.10.1.0/24 | 10.10.1.1 | Trunk native / Switch management |
| igb1.10 | 10 | 10.10.10.0/24 | 10.10.10.1 | Trusted users, PCs, phones, Wi-Fi |
| igb1.20 | 20 | 10.10.20.0/24 | 10.10.20.1 | IoT, printers, smart TVs, smart devices |
| igb1.30 | 30 | 10.10.30.0/24 | 10.10.30.1 | Guest Wi-Fi |
| igb1.40 | 40 | 10.10.40.0/24 | 10.10.40.1 | Cisco lab gear, fully isolated |
| igb1.50 | 50 | 10.10.50.0/24 | 10.10.50.1 | Management, direct WAN, no VPN |

---

## Physical Ports

| Port | Interface | Purpose | State |
|---|---|---|---|
| Port 1 | igb0 | WAN | Active |
| Port 2 | igb1 | LAN (trunk to GS308E) | Trunk (active) |
| Port 3 | igb2 | OOB_MGMT (out-of-band) | **Active** - subnet `172.16.99.0/24`, gateway `172.16.99.1`, DHCP `.10-.99`. Enterprise OOB recovery path. |
| Port 4 | igb3 | Lab Direct | Named, disabled |
| Port 5 | igb4 | High Availability (HA) | Named, disabled |
| Port 6 | igb5 | Expansion | Named, disabled |

---

## DHCP Ranges

| VLAN | Range | Notes |
|---|---|---|
| VLAN 1 | 10.10.1.10-10.10.1.99 | Switch management only |
| VLAN 10 | 10.10.10.10-10.10.10.200 | .201-.254 reserved (static / future use, .254 = AP) |
| VLAN 20 | 10.10.20.10-10.10.20.254 | |
| VLAN 30 | 10.10.30.10-10.10.30.200 | .201-.254 reserved (static / future use) |
| VLAN 40 | 10.10.40.10-10.10.40.254 | |
| VLAN 50 | 10.10.50.10-10.10.50.200 | .201-.254 reserved (static / future use) |

---

## Static DHCP Mappings

| Device | MAC | IP | VLAN |
|---|---|---|---|
| Netgear GS308E (Switch) | n/a | 10.10.1.100 | VLAN 1 |
| Netgear R6400 (AP) | n/a | 10.10.10.254 | VLAN 10 |

---

## Virtual IPs

Loopback service addresses. Not tied to any VLAN or physical interface. Used for internal services that need a reachable IP.

| Address | Interface | Purpose |
|---|---|---|
| 10.10.99.1/32 | Localhost | pfBlockerNG DNSBL sinkhole VIP |

> **Note:** 10.10.99.1 is a loopback IP Alias on the Localhost interface. It is not a VLAN gateway. Blocked DNS queries resolve to this address, and pfBlockerNG serves the block page from it over ports 80 and 443. The 10.10.99.0/24 range was chosen because it does not overlap with any existing VLAN (1, 10, 20, 30, 40, 50) or the Tailscale advertised routes (10.10.1.0/24 and 10.10.10.0/24).

# VLAN Assignments

Interface, subnet, and purpose map for the Router-on-a-Stick topology. All VLANs ride a single 802.1Q trunk on `igb1` (Port 7). Subnet third-octet always matches the VLAN ID.

---

## Interface Map

| Interface | VLAN | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| igb0 | - | DHCP/ISP | - | WAN |
| igb1 | 1 (Native) | 10.10.1.0/24 | 10.10.1.1 | Trunk native / Switch management |
| igb1.10 | 10 | 10.10.10.0/24 | 10.10.10.1 | Trusted users - PCs, phones, Wi-Fi |
| igb1.20 | 20 | 10.10.20.0/24 | 10.10.20.1 | IoT - printers, smart TVs, smart devices |
| igb1.30 | 30 | 10.10.30.0/24 | 10.10.30.1 | Guest Wi-Fi |
| igb1.40 | 40 | 10.10.40.0/24 | 10.10.40.1 | Cisco lab gear - fully isolated |
| igb1.50 | 50 | 10.10.50.0/24 | 10.10.50.1 | Management - direct WAN, no VPN |

---

## Physical Ports

| Port | Interface | Reserved For | State |
|---|---|---|---|
| Port 2 | igb1 | - | Trunk (active) |
| Port 3 | igb2 | Out-of-Band (OOB) | Named, disabled |
| Port 4 | igb3 | Lab Direct | Named, disabled |
| Port 5 | igb4 | High Availability (HA) | Named, disabled |
| Port 6 | igb5 | Expansion | Named, disabled |

---

## DHCP Ranges

| VLAN | Range | Notes |
|---|---|---|
| VLAN 1 | 10.10.1.10 - 10.10.1.254 | Switch management only |
| VLAN 10 | 10.10.10.10 - 10.10.10.253 | .254 reserved for AP static mapping |
| VLAN 20 | 10.10.20.10 - 10.10.20.254 | |
| VLAN 30 | 10.10.30.10 - 10.10.30.254 | |
| VLAN 40 | 10.10.40.10 - 10.10.40.254 | |
| VLAN 50 | 10.10.50.10 - 10.10.50.254 | |

---

## Static DHCP Mappings

| Device | MAC | IP | VLAN |
|---|---|---|---|
| Netgear GS308E (Switch) | - | 10.10.1.100 | VLAN 1 |
| Netgear R6400 (AP) | - | 10.10.10.254 | VLAN 10 |

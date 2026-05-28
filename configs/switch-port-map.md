# Switch Port Map

802.1Q port assignments for the **Netgear GS308E v4** smart-managed switch.

- **Management IP:** 10.10.1.100 (static, gateway 10.10.1.1)
- **Accessible via:** VLAN 50 (Management) only, VLAN 10 management access removed
- **Status:** Part 1 layout LOCKED. Any change requires a CHANGELOG entry.

---

## Port Assignments

| Port | Name | Mode | Native VLAN | Tagged VLAN IDs | Device |
|---|---|---|---|---|---|
| 1 | UPLINK-PFSENSE | Trunk | 1 (→ 999 planned) | 10 Trusted, 20 IoT, 30 Guest, 40 Lab, 50 Mgmt | pfSense igb1 (Protectli Port 2) |
| 2 | AP-TRUNK | Trunk | 10 Trusted | 30 Guest, 50 Mgmt | Ubiquiti U7 Lite (Wi-Fi 7) |
| 3 | GS305-CHAIN | Access | 10 Trusted | - | Netgear GS305 (unmanaged) → Kali laptop + VLAN 10 spare |
| 4 | MAC-MINI | Access | 10 Trusted | - | Mac Mini M4 (UniFi Network Controller host) |
| 5 | PERSONAL-PC | Access | 10 Trusted | - | Personal PC |
| 6 | PRINTER | Access | 20 IoT | - | Printer |
| 7 | SW1-CATALYST | Access | 40 Lab | - | Cisco Catalyst 3560 (CCNA lab uplink). R1 (1941 ISR) plugs into SW1 directly. |
| 8 | WAN-ESCAPE | Access | 50 Mgmt | - | Wired direct-WAN escape. Unplugged when not in use. |

> **Read the table as:** Native VLAN = the VLAN this port belongs to for untagged frames (= PVID). Tagged VLAN IDs = additional VLANs carried as 802.1Q tagged frames on the wire (trunks only). Access ports show `-` for Tagged VLAN IDs.

---

## VLAN Membership Summary

| VLAN | Members | Notes |
|---|---|---|
| 1 (default) | Port 1 (Native, no real members) | To be retired in favor of VLAN 999. |
| 10 Trusted | Port 1 (T), 2 (Native), 3 (U), 4 (U), 5 (U) | Trusted users. Port 2 carries it as the AP native VLAN. |
| 20 IoT | Port 1 (T), 6 (U) | Printer and any wired IoT. |
| 30 Guest | Port 1 (T), 2 (T) | Wireless-only via AP. No wired device on VLAN 30. |
| 40 Lab | Port 1 (T), 7 (U) | Cisco CCNA lab, isolated. |
| 50 Mgmt | Port 1 (T), 2 (T), 8 (U) | Management + wired WAN escape + NoVPN Wi-Fi. |
| 999 (planned) | Port 1 (Native, no other members) | Black-hole native VLAN for trunk hardening. |

---

## Wi-Fi to VLAN Mapping (via AP-TRUNK on Port 2)

| SSID | Tagging | VLAN |
|---|---|---|
| VPN_WiFi | Untagged (rides native) | 10 Trusted |
| Guest_WiFi | Tagged | 30 Guest |
| NoVPN_WiFi | Tagged | 50 Mgmt |

---

## Downstream Expansion

| Port | Downstream Device | Devices Behind It |
|---|---|---|
| 3 | Netgear GS305 (5-port unmanaged) | Kali laptop + spare VLAN 10 ports |
| 7 | Cisco Catalyst 3560 (8-port managed) | Cisco 1941 ISR (R1) + future CCNA lab gear |

> **Critical rule:** Unmanaged switches (GS305) **strip 802.1Q tags**. Never put a tagged-VLAN device behind GS305. The GS305 chain is VLAN 10 ONLY.

---

## Notes

- Port 1 (UPLINK-PFSENSE) is the only trunk to pfSense, carries all production VLANs.
- Port 2 (AP-TRUNK) is the second trunk, carries Wi-Fi VLANs (10 untagged, 30 + 50 tagged).
- Port 3 (GS305-CHAIN) feeds an unmanaged switch. VLAN 10 only, no tagging downstream possible.
- Ports 4 and 5 are dedicated VLAN 10 endpoints (Mac Mini, Personal PC). Daily-use workstations.
- Port 6 is VLAN 20 for the printer.
- Port 7 hands off to the Catalyst 3560 which is the CCNA lab fabric. R1 Cisco 1941 plugs into SW1, not into the GS308E directly.
- Port 8 is the wired WAN escape. Physical access = full WAN reach with no VPN. Keep unplugged when not in active use.

---

## GS308E Reassignment Rule (reminder)

Always: (1) Add port to NEW VLAN as U, (2) change PVID to NEW VLAN, (3) remove port from OLD VLAN. The switch refuses to remove a port from its current PVID VLAN.

---

## Pending Hardening (Part 1 close-out)

- Move native VLAN on Port 1 from `1` → `999` (create VLAN 999 with no members, set Port 1 PVID = 999). Verify pfSense sends all production traffic tagged on igb1 before flipping.
- Label all 8 ports in the GS308E UI (Port Description / Port Name field) with the names above.
- Export GS308E config and commit to repo for backup.

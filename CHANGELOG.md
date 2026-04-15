# Changelog

All notable changes to this project are documented here.

---

## [Phase 3] - In Progress

### Done
- Tailscale remote access - advertised routes for `10.10.1.0/24` and `10.10.10.0/24` configured and tested (March 2026)

### Fixed
- GS308E static DHCP moved from VLAN10 (wrong tab, wrong subnet) to LAN at `10.10.1.100`
- LAN DHCP range shrunk to `10.10.1.10 - 10.10.1.99` to avoid static mapping conflict
- VLAN 50 firewall docs updated from 2 rules to actual 5 rules
- VLAN 10 DHCP range corrected from .253 to .245 to match pfSense
- Clarified VLAN 50 WAN NAT behavior in docs
- Added temp switch management rule note to VLAN 10 firewall docs

### Exploring
- Suricata IDS / pfBlockerNG - evaluating options for alert-only monitoring on WAN and user VLANs
- Centralized logging - syslog server for firewall and IDS alerts

### On Hold
- AP hardware upgrade - VLAN-aware AP (TP-Link EAP or UniFi) not planned right now, may revisit
- Guest rate limiting on VLAN 30 - deferred due to R6400 hardware limitations

---

## [Phase 2] - 2026-03-17

### Added
- Router-on-a-Stick topology via single 802.1Q trunk (`igb1`)
- 6 VLANs: Native (1), Users (10), IoT (20), Guest (30), Lab (40), Management (50)
- Dual Mullvad WireGuard tunnels - Chicago (primary) + NYC (failover)
- `VPN_FAILOVER` gateway group with automatic failover
- Layered kill switch - manual outbound NAT with zero WAN rules, DoH/DoT block, port 53 block, RFC1918 inter-VLAN isolation, IPv6 block
- DNS locked to Mullvad resolvers through VPN tunnels - no ISP fallback
- Netgear GS308E v4 configured with 802.1Q port assignments
- Netgear R6400 set to AP-only mode on VLAN 10
- Cisco lab gear isolated on VLAN 40
- Leak testing passed - ipleak.net and Mullvad connection check

### Changed
- Switch management moved to VLAN 50 only - removed VLAN 10 access

### Documented
- Full architecture writeup, firewall rules manual, and switch/VLAN manual (PDFs in `/docs`)
- Shareable config breakdowns in `/configs`

---

## [Phase 1] - Early 2026

### Added
- Initial pfSense 2.8.1 install on Protectli FW6E
- Basic WAN + LAN configuration
- Single Mullvad WireGuard tunnel (Chicago)
- Early VLAN segmentation
- Initial firewall rules and NAT setup

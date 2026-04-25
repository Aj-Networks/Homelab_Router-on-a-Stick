# Changelog

All notable changes to this project are documented here.

---

## [Phase 3] - In Progress

### Done
- Tailscale remote access - advertised routes for `10.10.1.0/24` and `10.10.10.0/24` configured and tested (March 2026)
- Suricata IDS deployed on pfSense 2.8.1
- pfBlockerNG-devel deployed for IP/DNS blocking
- pfBlockerNG DNSBL VIP activated on 10.10.99.1/32 Localhost (April 2026). Network wide ad and tracker DNS blocking now active on VLAN10_USERS, VLAN20_IOT, VLAN30_GUEST, and VLAN50_MGMT. Blocked lookups resolve to the VIP and serve the pfBlockerNG block page.
- pfBlockerNG DNSBL groups expanded (April 2026): ADs_Basic, ADs, Firebog_Suspicious, Phishing, BBcan177, Malicious - all set to Unbound action and daily updates. Verified `nslookup doubleclick.net` returns the DNSBL VIP.
- VLAN 10 firewall rule #1 narrowed (April 2026) from blanket `Pass any -> 10.10.10.1` to three scoped rules: UDP 53 (DNS), TCP 443 (WebUI), ICMP (gateway reachability).
- VLAN 50 firewall rule #1 narrowed (April 2026) from blanket `Pass any -> 10.10.50.1` to four scoped rules: UDP 53, TCP 443, TCP 22 (SSH), ICMP. Rule #7 `Pass any -> any` retained for the escape-hatch workflow.
- Tailscale ACLs locked down (April 2026): subnet approval reduced to `10.10.10.0/24` only, default-deny ACL with `tag:home` + `autogroup:admin` -> `10.10.10.0/24:*`, primary device tagged `tag:home`. VLAN 50 stays physical-only via switch port 8.
- Suricata enabled on VLAN 40 (April 2026) in alert-only mode. Suppression list deferred until lab gear comes online and Cisco-protocol SIDs (CDP, STP, DTP) can be observed.
- Backup procedure documented (April 2026) - pfSense XML export and GS308E config backup with age encryption (`configs/backup-procedure.md`).
- Repeatable testing procedures documented (April 2026) - VLAN isolation matrix, DNSBL verify, kill-switch drill, failover drill, leak tests (`configs/testing-procedures.md`).

### Fixed
- GS308E static DHCP moved from VLAN10 (wrong tab, wrong subnet) to LAN at `10.10.1.100`
- LAN DHCP range shrunk to `10.10.1.10 - 10.10.1.99` to avoid static mapping conflict
- VLAN 50 firewall docs updated from 2 rules to actual 5 rules
- VLAN 10 DHCP range corrected from .253 to .245 to match pfSense
- Clarified VLAN 50 WAN NAT behavior in docs
- Added temp switch management rule note to VLAN 10 firewall docs
- VLAN 10 rule #1 updated to pass any (pfSense UI access for lab work)
- VLANs 20, 30, 40 rule #1 locked down to UDP 53 only (DNS)
- DoH/DoT port range corrected from "443, 853" to "443-853" across all VLANs
- Added missing LAN (VLAN 1) firewall rules section
- Added hardware images to repo structure diagram
- Renamed "Reserved For" column to "Purpose" in physical ports table

### Exploring
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
- Layered kill switch - manual outbound NAT with zero explicit WAN rules, DoH/DoT block, port 53 block, RFC1918 inter-VLAN isolation, IPv6 block
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

## [Phase 0] - 2024 Origins

### Added
- Early 2024 - Acquired Protectli FW6E, initial pfSense install with basic WAN/LAN configuration
- Later 2024 - First Mullvad WireGuard tunnel (Chicago), started learning VPN enforcement and outbound NAT discipline
- Through 2025 - Added DNS and DoH blocking, built up early kill switch rules incrementally, refined VLAN segmentation
- Late 2025 - Work transitioned into the polished Phase 2 build (dual tunnel failover, 6 VLAN topology, Tailscale, Suricata, pfBlockerNG-devel)

> **Note:** Phase 0 absorbs the original Phase 1 scope. Earlier drafts tagged this same work as "Phase 1 - Early 2026" with incorrect dates. The real timeline is 2024 onward.

<div align="center">

# Homelab: pfSense Router-on-a-Stick

**Privacy-first home network. 6 VLANs, dual WireGuard failover, fail-closed by design.**

![pfSense](https://img.shields.io/badge/pfSense-2.8.1-orange?logo=pfsense&logoColor=white)
![UniFi](https://img.shields.io/badge/UniFi-U7%20Lite-blue?logo=ubiquiti&logoColor=white)
![Mullvad](https://img.shields.io/badge/VPN-Mullvad%20WireGuard-yellow)
![Status](https://img.shields.io/badge/Lab-Active-brightgreen)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)
![Last commit](https://img.shields.io/github/last-commit/Aj-Networks/Homelab_Router-on-a-Stick)

</div>

No traffic leaves this network outside an encrypted tunnel. The kill switch is the **absence of WAN egress NAT**, not a block rule, so a leak path would have to be created rather than merely permitted. Every VLAN routes through the firewall, which is what makes default-deny segmentation enforceable.

Configuration is documented with its reasoning. Failures are documented in [LIMITATIONS.md](docs/LIMITATIONS.md) with root cause and reattempt criteria.

---

## Specification

| | |
|---|---|
| Firewall | Protectli FW6E, pfSense 2.8.1 |
| Switch | Netgear GS308E v4, 802.1Q trunk |
| Wi-Fi | UniFi U7 Lite, 3 VLAN-tagged SSIDs |
| VPN | Mullvad WireGuard, 2 tunnels, gateway-group failover |
| IDS / DNS filter | Suricata (6 interfaces), pfBlockerNG-devel |
| Overlay | Tailscale, ACL-scoped to one subnet |
| Application host | Mac Mini M4, Docker |
| Practice lab | Cisco Catalyst 3560 + 1941 ISR, isolated on VLAN 40 |
| Build cost | ~$1,960 |

---

## Topology

<p align="center">
  <img src="assets/diagrams/network-topology.png" alt="Network topology" width="100%"/>
</p>

### Segmentation

<p align="center">
  <img src="assets/diagrams/vlan-ip-detail.png" alt="VLAN and IP detail" width="100%"/>
</p>

Third octet matches the VLAN ID. `10.10.20.5` is unambiguously a VLAN 20 host.

| VLAN | Purpose | Egress |
|---|---|---|
| 10 | Trusted users | VPN tunnel |
| 20 | IoT | VPN tunnel |
| 30 | Guest | VPN tunnel, no LAN access |
| 40 | Cisco lab | Isolated, excluded from IDS |
| 50 | Management | **Direct WAN, no tunnel.** No admin rights, no inter-VLAN reach |

---

## Egress controls

Seven layers. A packet must defeat all of them to leave unencrypted.

| # | Control | Mechanism |
|---|---|---|
| 1 | NAT lock | Outbound NAT bound to WireGuard interfaces. One WAN NAT rule exists, for VLAN 50 |
| 2 | Encrypted DNS block | DoH and DoT to known resolvers blocked on 443 and 853 |
| 3 | Plain DNS block | Port 53 to WAN blocked for all clients |
| 4 | Inter-VLAN deny | RFC1918 blocked between segments, allow-listed per rule |
| 5 | IPv6 drop | All IPv6 dropped. A dual-stack kill switch doubles rule surface with silent failure modes |
| 6 | Resolver lock | Unbound forwards only through the active VPN gateway |
| 7 | Fail closed | Both tunnels down means traffic drops. No WAN fallback |

VLAN 50 is the deliberate exception: direct WAN so the firewall stays reachable when the tunnels themselves are the fault. It holds no administrative privileges.

---

## Documentation

**Network**

| Document | Scope |
|---|---|
| [vlan-assignments.md](network/vlan-assignments.md) | Interface, VLAN and subnet map |
| [switch-port-map.md](network/switch-port-map.md) | Port allocation, AP trunk tagging |
| [firewall-rules.md](network/firewall-rules.md) | Per-VLAN rule chains and ordering |
| [nat-rules.md](network/nat-rules.md) | Outbound NAT design behind the kill switch |

**VPN and DNS**

| Document | Scope |
|---|---|
| [vpn-failover.md](vpn/vpn-failover.md) | Tunnels, gateway groups, build checklist, post-build verification |
| [dns-resilience.md](vpn/dns-resilience.md) | Four-layer resilience, including the parts later disproven under test |

**Services**

| Document | Scope |
|---|---|
| [pfblockerng.md](services/pfblockerng.md) | DNSBL groups, sinkhole VIP, update policy |
| [tailscale.md](services/tailscale.md) | Routes, ACLs, tag policy |
| [mac-mini/](services/mac-mini/) | Docker host, remote access |

**Operations**

| Document | Scope |
|---|---|
| [backup-procedure.md](operations/backup-procedure.md) | Encrypted export and restore |
| [testing-procedures.md](operations/testing-procedures.md) | Isolation matrix, kill-switch drill, failover drill, leak tests |

**Reference**

| Document | Scope |
|---|---|
| [LIMITATIONS.md](docs/LIMITATIONS.md) | Post-mortems: hardware ceilings, design constraints, root causes |
| [technical-guide.md](docs/technical-guide.md) | 29-section build writeup |
| [docs/exports/](docs/exports/) | PDF and DOCX exports |
| [labs/ccna/](labs/ccna/) | Cisco practice lab, VLAN 40 |
| [archive/](archive/) | Retired hardware documentation |

---

## Verification

Procedures in [testing-procedures.md](operations/testing-procedures.md).

| Test | Result |
|---|---|
| IP, DNS and WebRTC leak | Clean. [ipleak.net](assets/screenshots/ipleak.png), [Mullvad Check](assets/screenshots/mullvad.png) |
| VPN failover | Tier 1 disabled, Tier 2 promoted, no egress during transition |
| Inter-VLAN isolation | Cross-VLAN reachability blocked per matrix |
| Kill switch | Both tunnels down, all client traffic dropped, no WAN fallback |

---

## Limitations

| Limitation | Detail |
|---|---|
| Single firewall | No HA pair. `igb4` reserved for a future CARP peer |
| Switch tier | GS308E v4 has no management VLAN, no ACLs, no SSH. Caps trunk hardening. [Post-mortem](docs/LIMITATIONS.md) |
| IDS blocking is global | The Suricata block table applies firewall-wide, not per interface. [Post-mortem](docs/LIMITATIONS.md) |

---

## Recent work

| Date | Change |
|---|---|
| **2026-08-06** | Management VLAN stripped of admin rights. It is broadcast as an SSID and carried pass rules for the firewall WebUI, SSH, and unrestricted inter-VLAN reach, so anyone with the Wi-Fi password held all three from outside the building. A firewall matches on source subnet and cannot distinguish wired from wireless on the same VLAN. Rules removed, RFC1918 blocked, out-of-band access verified reachable first. |
| **2026-08-06** | Orphaned outbound NAT rules found after a tunnel replacement. Deleting a VPN interface leaves its NAT rules in place pointing at nothing, still rendering as valid in the UI. Three VLANs had no internet for an unknown period. Verification moved to `pfctl -sn` against the running packet filter. |
| **2026-08-06** | Wi-Fi throughput on the trusted VLAN restored from 27 Mbps to 519 Mbps. An 802.1Q mismatch, the controller tagging a VLAN the switch port carries untagged, pushed the access point off hardware forwarding into software bridging. Traffic passed correctly and every RF metric read healthy, so nothing alarmed. Found by testing a second SSID on a different VLAN, same AP, same moment. |
| **2026-08-06** | Six weeks of IDS false positives closed at root cause. One ruleset generated roughly 99.5% of ~980,000 alerts by flagging normal VPN-tunnel TCP behaviour. Disabled lab-wide, replaced with curated known-bad-indicator rulesets in detect-only mode. |
| **2026-08-06** | Three latent faults on rebuilt WireGuard interfaces: a gateway monitoring its own interface address so failover could never fire, a missing MTU clamp causing a PMTU black hole, and a DNS forwarder with no route. Distilled into a build checklist in [vpn-failover.md](vpn/vpn-failover.md). |

Full history in [CHANGELOG.md](CHANGELOG.md).

---

## Roadmap

| Item | Status |
|---|---|
| AP upgrade to Wi-Fi 7 | Done, May 2026 |
| Guest VLAN tagging over trunk | Done, May 2026 |
| Dedicated out-of-band management port | Done, May 2026 |
| IDS ruleset promotion to blocking interfaces | Gated on a clean detect-only period |
| Egress filtering on the IoT VLAN | Planned |
| Managed switch replacement | Planned. Unblocks native VLAN 999 and a dedicated management VLAN |
| Native VLAN 999 and management VLAN | Blocked by current switch. [Detail](docs/LIMITATIONS.md) |
| CCNA exam, VLAN 40 practice lab | Scheduled, October 2026 |
| HA firewall pair | Under consideration |

---

## Background

System administrator and network engineer, 7+ years across healthcare, education and MSP environments. Active Directory, Intune, pfSense, HIPAA compliance.

This lab exists to test ideas properly before they inform production work: default-deny segmentation, verifiable zero-leak defaults, and separation of firewall and application duties.

[ajayangdembe.com](https://www.ajayangdembe.com)

---

## License

MIT. See [LICENSE](LICENSE).

Use and adapt freely. If you republish substantial parts, keep the copyright notice and link back, as the license requires.

<div align="center">

*[Aj-Networks](https://github.com/Aj-Networks)*

</div>

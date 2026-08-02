<div align="center">

# Homelab: pfSense Router-on-a-Stick

**A privacy-first home network built from off-the-shelf gear.**
*6 VLANs · dual VPN failover · layered kill switch · verified zero leaks*

![pfSense](https://img.shields.io/badge/pfSense-2.8.1-orange?logo=pfsense&logoColor=white)
![UniFi](https://img.shields.io/badge/UniFi-U7%20Lite-blue?logo=ubiquiti&logoColor=white)
![Mullvad](https://img.shields.io/badge/VPN-Mullvad%20WireGuard-yellow)
![Status](https://img.shields.io/badge/Lab-Active-brightgreen)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)
![Last commit](https://img.shields.io/github/last-commit/Aj-Networks/Homelab_Router-on-a-Stick)

</div>

A single-firewall home network where no traffic leaves the house outside an encrypted tunnel. Six VLANs with default-deny segmentation, dual WireGuard tunnels with automatic failover, and a kill switch built from the absence of WAN egress NAT rather than a rule that could be misordered. Every design decision is documented with its reasoning, and every failure is written up in [LIMITATIONS.md](docs/LIMITATIONS.md).

---

## At a glance

| | |
|---|---|
| **VLANs** | 6 (Trusted, IoT, Guest, Lab, Mgmt, Native) |
| **VPN tunnels** | 2 (Mullvad WireGuard, dual-region, automatic failover) |
| **Kill switch layers** | 7 (NAT, DoH/DoT block, port 53 block, RFC1918 block, IPv6 block, DNS lockdown, no WAN egress NAT) |
| **Leak testing** | DNS, IP, WebRTC verified clean (ipleak.net, Mullvad Check) |
| **IDS** | Suricata on 6 interfaces, curated known-bad rulesets |
| **Hardware cost** | ~$1,960 |

---

## Architecture

<p align="center">
  <img src="assets/diagrams/network-topology.png" alt="Network topology" width="100%"/>
</p>

### Stack

| Layer | Tool |
|---|---|
| Firewall + Router | Protectli FW6E, pfSense 2.8.1 |
| Switch | Netgear GS308E v4 (802.1Q trunking) |
| Wi-Fi | UniFi U7 Lite (WiFi 7, VLAN-tagged SSIDs) |
| VPN | Mullvad WireGuard, dual-tunnel failover group |
| IDS / DNS filtering | Suricata + pfBlockerNG-devel |
| Overlay / remote access | Tailscale (scoped ACLs) |
| Application host | Mac Mini M4, 24/7 Docker host |
| Practice lab | Cisco Catalyst 3560 + Cisco 1941 ISR, isolated on VLAN 40 |

### VLAN and IP plan

<p align="center">
  <img src="assets/diagrams/vlan-ip-detail.png" alt="VLAN and IP detail" width="100%"/>
</p>

Subnet convention: the third octet matches the VLAN ID, so logs read at a glance.

---

## Privacy design

Seven independent defenses. A packet must bypass all of them to leak.

1. **NAT lock.** Outbound NAT is bound to the VPN interfaces only. No WAN egress NAT rules exist, so a leak path would have to be created, not merely allowed.
2. **Encrypted-DNS block.** DoH/DoT egress (443/853 to known resolvers) is blocked; applications cannot bypass the resolver.
3. **Plain-DNS block.** Port 53 to WAN is blocked for all clients.
4. **Inter-VLAN default deny.** RFC1918 space is blocked between segments; each VLAN sees only what it is explicitly allowed.
5. **IPv6 dropped.** No v6 leak vector until the design covers it deliberately.
6. **DNS inside the tunnel.** The resolver forwards only through the active VPN gateway; the ISP sees encrypted UDP and nothing else.
7. **Fail closed.** Both tunnels down means all traffic drops. No fallback to WAN, no silent failure.

The one deliberate exception: VLAN 50 egresses directly, providing an admin path to the firewall when the tunnels themselves are the problem.

---

## Documentation

Each document covers one concern and explains the reasoning, not just the configuration.

### Network design

| Doc | Scope |
|---|---|
| [vlan-assignments.md](network/vlan-assignments.md) | Interface, VLAN and subnet map |
| [switch-port-map.md](network/switch-port-map.md) | GS308E port allocation (locked layout) |
| [firewall-rules.md](network/firewall-rules.md) | Per-VLAN rule chains |
| [nat-rules.md](network/nat-rules.md) | The outbound NAT design behind the kill switch |

### VPN and DNS

| Doc | Scope |
|---|---|
| [vpn-failover.md](vpn/vpn-failover.md) | WireGuard tunnels, gateway groups, tunnel build checklist and post-build verification |
| [dns-resilience.md](vpn/dns-resilience.md) | Four-layer DNS resilience, including what failed and was corrected |

### Services

| Doc | Scope |
|---|---|
| [pfblockerng.md](services/pfblockerng.md) | DNSBL groups and update policy |
| [tailscale.md](services/tailscale.md) | Overlay routes, ACLs, tag policy |
| [mac-mini/](services/mac-mini/) | Docker host setup and remote access |

### Operations

| Doc | Scope |
|---|---|
| [backup-procedure.md](operations/backup-procedure.md) | Encrypted config backup and restore |
| [testing-procedures.md](operations/testing-procedures.md) | Repeatable verification: isolation matrix, kill-switch drill, failover drill, leak tests |

### Reference

| Doc | Scope |
|---|---|
| [LIMITATIONS.md](docs/LIMITATIONS.md) | Post-mortems: hardware ceilings, design constraints, incidents and their root causes |
| [technical-guide.md](docs/technical-guide.md) | Full 29-section writeup of the build, aimed at readers learning the stack |
| [docs/exports/](docs/exports/) | PDF and DOCX exports of the architecture and manuals |
| [labs/ccna/](labs/ccna/) | Cisco practice lab (CCNA preparation, isolated on VLAN 40) |
| [archive/](archive/) | Docs for retired hardware, kept for reference |

---

## Verification

Claims above are tested, not assumed. Procedures live in [testing-procedures.md](operations/testing-procedures.md).

- Zero IP / DNS / WebRTC leaks: [ipleak.net](assets/screenshots/ipleak.png) · [Mullvad Check](assets/screenshots/mullvad.png)
- VPN failover: primary tunnel disabled, secondary promoted, no traffic escapes during the transition
- Inter-VLAN isolation: cross-VLAN reachability tested per the isolation matrix
- Kill switch: both tunnels down, all client traffic drops, no WAN fallback

---

## Known limitations

- **Single firewall, single point of failure.** No HA pair; an accepted trade-off at home scale.
- **Switch hardware ceiling.** The GS308E v4 has no management-VLAN capability, which caps trunk hardening below the enterprise pattern. Full post-mortem and reattempt criteria in [LIMITATIONS.md](docs/LIMITATIONS.md).
- **IDS blocking is global, not per-interface.** Suricata's block table applies firewall-wide; the constraint and the operating policy that came out of it are documented in [LIMITATIONS.md](docs/LIMITATIONS.md).

---

## Recent work

| Date | Change |
|---|---|
| **2026-07-31** | Wi-Fi throughput on the trusted VLAN restored from 27 Mbps to 519 Mbps. An 802.1Q tagging mismatch on the AP uplink (controller tagging a VLAN the switch port carries untagged) pushed the access point off its hardware forwarding path into software bridging. Traffic passed correctly throughout and every RF metric read healthy, so nothing ever alarmed. Found by testing a client on a second SSID on a different VLAN, same AP and same moment. Guidance in [switch-port-map.md](network/switch-port-map.md). |
| **2026-07-31** | Six weeks of IDS false positives closed at root cause. Measuring the alert distribution showed one ruleset (`stream-events.rules`) generating ~99.5% of ~980k alerts by flagging normal VPN-tunnel TCP behavior; it is now disabled lab-wide and replaced with curated known-bad-indicator rulesets rolled out detect-first. |
| **2026-07-31** | Three latent faults found and fixed on rebuilt WireGuard interfaces: a self-referencing gateway monitor that made failover impossible, a missing MTU clamp causing a PMTU black hole, and a DNS forwarder with no route. Distilled into a tunnel build checklist in [vpn-failover.md](vpn/vpn-failover.md). |
| **2026-06-04** | DNS resilience rebuilt as a four-layer defense after a resolver soft-hang took down LAN DNS. Design later corrected when parts of it were disproven under test; both versions are documented in [dns-resilience.md](vpn/dns-resilience.md). |

Full history in [CHANGELOG.md](CHANGELOG.md).

---

## Roadmap

| Item | Status |
|---|---|
| AP hardware upgrade (U7 Lite) | Done, May 2026 |
| Guest VLAN tagging over trunk | Done, May 2026 |
| Dedicated OOB management port | Done, May 2026 |
| IDS threat-ruleset rollout to blocking interfaces | In progress, gated on a clean detect-only week |
| Native VLAN 999 + dedicated mgmt VLAN | Blocked by switch hardware, see [LIMITATIONS.md](docs/LIMITATIONS.md) |
| CCNA exam on the VLAN 40 practice lab | Scheduled, August 2026 |
| Centralized syslog | Evaluating |

---

## Background

Built and maintained by a system administrator / network engineer with 7+ years across healthcare, education, and MSP environments (Active Directory, Intune, pfSense, HIPAA compliance). The lab exists to test ideas properly before they inform production work: default-deny segmentation, verifiable zero-leak defaults, and separation of firewall and application duties.

More projects: [ajayangdembe.com](https://www.ajayangdembe.com)

---

## License

MIT. See [LICENSE](LICENSE).

Use, fork, and build on this freely. If you republish substantial parts, keep the copyright notice and a link back to this repository, as the license requires.

<div align="center">

*Built by [Aj-Networks](https://github.com/Aj-Networks)*

</div>

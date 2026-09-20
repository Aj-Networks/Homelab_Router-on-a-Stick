<div align="center">

# Homelab: pfSense Router-on-a-Stick

**A home network where nothing reaches the internet outside an encrypted tunnel.**

![pfSense](https://img.shields.io/badge/pfSense-2.8.1-orange?logo=pfsense&logoColor=white)
![UniFi](https://img.shields.io/badge/UniFi-U7%20Lite-blue?logo=ubiquiti&logoColor=white)
![Mullvad](https://img.shields.io/badge/VPN-Mullvad%20WireGuard-yellow)
![Status](https://img.shields.io/badge/Lab-Active-brightgreen)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)
![Last commit](https://img.shields.io/github/last-commit/Aj-Networks/Homelab_Router-on-a-Stick)

</div>

Six isolated network segments, two VPN tunnels with automatic failover, and a kill switch that works by **leaving something out** rather than blocking it: there are no rules permitting unencrypted traffic to reach the internet, so a leak would have to be deliberately created.

Every design choice is documented with its reasoning. Every failure is written up with its root cause.

---

## Recent work

| Date | Change |
|---|---|
| **2026-09-20** | Intrusion detection was silently blocking legitimate services (Apple, Meta, Cloudflare) after a configuration restore quietly undid an earlier fix. It ran unnoticed for six weeks, appearing as intermittent Wi-Fi slowness. Blocking disabled, and an hourly check added so a silent reversion cannot happen again. |
| **2026-08-06** | Wi-Fi on the main network ran at 27 Mbps instead of 519. The access point and the switch disagreed about how one network segment was labelled, which pushed the AP into a slow software path. Every signal metric read perfectly healthy, so nothing alarmed. Found by comparing two Wi-Fi networks on the same radio at the same moment. |

<details>
<summary><b>See more</b></summary>

| Date | Change |
|---|---|
| **2026-08-06** | Removed administrative access from the management segment. It is broadcast as a Wi-Fi network, so anyone with that password reached the firewall's admin interface and every device on the network, from outside the building. |
| **2026-08-06** | Found network address translation rules left pointing at a deleted VPN tunnel. Three segments had no internet for an unknown period while the rest of the network behaved normally. |
| **2026-08-06** | Six weeks of false security alerts traced to a single rule set producing 99.5% of roughly 980,000 alerts, by flagging normal VPN behaviour as suspicious. Replaced with rules that match known threats instead. |
| **2026-08-06** | Three hidden faults on rebuilt VPN tunnels: a health check watching itself so failover could never trigger, a missing packet-size limit, and a DNS server with no route to reach it. |
| **2026-06-04** | Rebuilt DNS resilience after the resolver silently stopped answering. Parts of that design were later disproven under test and corrected. |

</details>

Full history in [CHANGELOG.md](CHANGELOG.md).

---

## Hardware and software

| | |
|---|---|
| Firewall | Protectli FW6E, pfSense 2.8.1 |
| Switch | Netgear GS308E v4 |
| Wi-Fi | UniFi U7 Lite, three separate wireless networks (upgraded from a Netgear R6400 in May 2026) |
| VPN | Mullvad WireGuard, two tunnels with automatic failover |
| Security monitoring | Suricata intrusion detection, pfBlockerNG DNS filtering |
| Remote access | Tailscale, restricted to one segment |
| Applications | Mac Mini M4 running Docker |
| Practice lab | Cisco Catalyst 3560 and 1941 router, fully isolated |
| Build cost | $3,999+ one time, plus ~$199/year recurring |

> **This does not need to cost $4,000.** The same design runs on **$499 to $999** if you buy the firewall and switch used, keep an access point you already own, and use the free software this lab runs on: pfSense, Suricata, pfBlockerNG, WireGuard and Tailscale all cost nothing. The hardware here reflects choices made over two years for headroom and for learning, not the minimum to reproduce it.

---

## Network design

<p align="center">
  <img src="assets/diagrams/network-topology.png" alt="Network topology" width="100%"/>
</p>

Traffic is split into six segments. Devices in one segment cannot reach another unless a rule explicitly allows it.

| Segment | Holds | Internet path |
|---|---|---|
| 10 | Trusted computers and phones | VPN tunnel |
| 20 | Smart devices and printer | VPN tunnel |
| 30 | Guests | VPN tunnel, no access to anything local |
| 40 | Cisco practice lab | VPN tunnel. Cannot reach any other segment |
| 50 | Management | Direct, no VPN. Holds no admin rights |

<details>
<summary><b>Addressing detail</b></summary>

<p align="center">
  <img src="assets/diagrams/vlan-ip-detail.png" alt="VLAN and IP detail" width="100%"/>
</p>

Each segment's address range matches its ID, so `10.10.20.5` is recognisably a segment 20 device at a glance in any log.

</details>

---

## How traffic is kept inside the tunnel

Seven independent controls. A packet has to defeat all of them to leave unencrypted.

| Control | What it does |
|---|---|
| Address translation lock | Only VPN tunnels can rewrite addresses for internet-bound traffic. One exception exists, for the management segment |
| Encrypted DNS block | Applications cannot use their own hidden DNS to bypass the firewall's |
| Plain DNS block | Devices cannot query outside DNS servers directly |
| Segment isolation | Private address ranges are blocked between segments by default |
| IPv6 dropped | Supporting both address types doubles the rules and adds silent failure modes |
| Resolver lock | DNS queries leave only through the active VPN tunnel |
| Fail closed | If both tunnels drop, traffic stops. There is no fallback to an unencrypted path |

Segment 50 is the deliberate exception. It reaches the internet directly so the firewall stays reachable when the tunnels themselves are the problem, and it carries no administrative privileges.

---

## Verification

Claims here are tested. Procedures in [testing-procedures.md](operations/testing-procedures.md).

| Test | Result |
|---|---|
| IP, DNS and browser address leaks | None detected. [Evidence](assets/screenshots/ipleak.png) |
| VPN failover | Primary disabled, secondary took over, nothing escaped during the switch |
| Segment isolation | Cross-segment access blocked as designed |
| Kill switch | Both tunnels down, all traffic stopped, no fallback |

---

## Known limitations

| Limitation | Detail |
|---|---|
| One firewall | No redundant pair. A port is reserved for a future second unit |
| Switch capability | This model cannot separate management traffic onto its own segment, which caps how far the design can be hardened. [Post-mortem](docs/LIMITATIONS.md) |
| Security blocking is network-wide | The intrusion detection block list applies everywhere at once, not per segment. This caused four outages and is documented in full. [Post-mortem](docs/LIMITATIONS.md) |

---

## Documentation

<details>
<summary><b>Full index</b></summary>

**Network**

| Document | Scope |
|---|---|
| [vlan-assignments.md](network/vlan-assignments.md) | Interface, segment and address map |
| [switch-port-map.md](network/switch-port-map.md) | Port allocation and access point trunk |
| [firewall-rules.md](network/firewall-rules.md) | Rule chains per segment, and ordering |
| [nat-rules.md](network/nat-rules.md) | The address translation design behind the kill switch |

**VPN and DNS**

| Document | Scope |
|---|---|
| [vpn-failover.md](vpn/vpn-failover.md) | Tunnels, failover groups, build checklist |
| [dns-resilience.md](vpn/dns-resilience.md) | Four-layer resilience, including what was later disproven |

**Services**

| Document | Scope |
|---|---|
| [pfblockerng.md](services/pfblockerng.md) | DNS filtering groups and update policy |
| [tailscale.md](services/tailscale.md) | Remote access routes and restrictions |
| [mac-mini/](services/mac-mini/) | Application host setup |

**Operations**

| Document | Scope |
|---|---|
| [backup-procedure.md](operations/backup-procedure.md) | Encrypted export and restore |
| [testing-procedures.md](operations/testing-procedures.md) | Isolation, kill switch, failover and leak tests |

**Reference**

| Document | Scope |
|---|---|
| [LIMITATIONS.md](docs/LIMITATIONS.md) | Post-mortems with root causes and reattempt criteria |
| [technical-guide.md](docs/technical-guide.md) | Full 29-section build writeup |
| [docs/exports/](docs/exports/) | PDF and Word exports |
| [labs/ccna/](labs/ccna/) | Cisco practice lab |
| [archive/](archive/) | Retired hardware |

</details>

---

## Roadmap

<details>
<summary><b>Planned and completed</b></summary>

| Item | Status |
|---|---|
| Wi-Fi 7 access point | Done, May 2026 |
| Guest network on its own segment | Done, May 2026 |
| Dedicated recovery port on the firewall | Done, May 2026 |
| Threat rules on the internal segments | Next, in detection-only first |
| Outbound filtering for smart devices | Planned |
| Managed switch replacement | Planned. Unblocks the remaining hardening |
| Redundant firewall pair | Under consideration |
| CCNA exam | Scheduled, October 2026 |

</details>

---

## Background

System administrator and network engineer, 7+ years across healthcare, education and MSP environments. Active Directory, Intune, pfSense, HIPAA compliance.

This lab is where ideas get tested properly before they inform production work.

[ajayangdembe.com](https://www.ajayangdembe.com)

---

## License

MIT. See [LICENSE](LICENSE). Use and adapt freely; keep the copyright notice and a link back if you republish substantial parts.

<div align="center">

*[Aj-Networks](https://github.com/Aj-Networks)*

</div>

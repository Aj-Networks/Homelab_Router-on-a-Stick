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

Six isolated segments, two VPN tunnels with automatic failover, and a kill switch built from the **absence** of a rule rather than the presence of one. There are no rules permitting unencrypted traffic to reach the internet, so a leak would have to be deliberately created rather than merely allowed.

The interesting part of this repository is not the configuration. It is what happened when the configuration was wrong.

---

## How this lab is run

Four practices, each adopted after something failed.

**Verify against the stored configuration, not the interface.**
Twice, the web interface displayed the intended state while the underlying configuration said otherwise. Verification now reads the config file directly. One command replaced an hour of screenshots.

**Anything silently reversible needs a detector.**
Three separate fixes were correct when applied and were later undone without notice, one running six weeks before anyone noticed. The fourth attempt added an hourly check that writes to the system log, tested against a reserved address before being relied upon. A fix that can be silently reverted is not complete until something detects the reversion.

**Measure the distribution before chasing the signature.**
Six weeks of security incidents ended when the alert log was counted rather than read: one rule set produced 99.5% of roughly 980,000 alerts, by flagging normal VPN behaviour as suspicious. One command answered in seconds what six weeks of investigation had not.

**Accept risk explicitly, with criteria for revisiting it.**
Where a control is absent, the threat model, the compensating controls, and the conditions that would change the decision are written down. See [LIMITATIONS.md](docs/LIMITATIONS.md) section 3 for a worked example.

Failures are documented with root cause and reattempt criteria, including the ones that were my own mistakes. [LIMITATIONS.md](docs/LIMITATIONS.md) exists for that purpose.

---

## Recent work

| Date | Change |
|---|---|
| **2026-09-20** | Intrusion detection was silently blocking legitimate services after a configuration restore undid an earlier fix. It ran unnoticed for six weeks, appearing only as intermittent Wi-Fi slowness, because blocked packets are dropped silently and clients wait out a timeout rather than failing fast. Blocking disabled, and an hourly detector added. |
| **2026-08-06** | Wi-Fi on the trusted segment ran at 27 Mbps instead of 519. The access point and the switch disagreed about how one segment was labelled, pushing the AP into a slow software path. Every signal metric read healthy, so nothing alarmed. Found by comparing two wireless networks on the same radio at the same moment. |

<details>
<summary><b>See more</b></summary>

| Date | Change |
|---|---|
| **2026-09-20** | Wireless raised to WPA2/WPA3 with Protected Management Frames, closing a path from radio range to the trusted segment. Configuration backups encrypted at rest; they had held tunnel private keys and password hashes in a cloud-synchronised folder. |
| **2026-08-06** | Removed administrative access from the management segment. It is broadcast as a wireless network, so anyone with that password reached the firewall interface and every device, from outside the building. |
| **2026-08-06** | Found address translation rules still pointing at a deleted tunnel. Three segments had no internet for an unknown period while everything else behaved normally. |
| **2026-08-06** | Three latent faults on rebuilt tunnels: a health check watching itself so failover could never trigger, a missing packet-size limit, and a DNS server with no route to reach it. |
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

> **This does not need to cost $4,000.** The same design runs on **$499 to $999** with a used firewall and switch, an access point you already own, and the free software this lab runs on: pfSense, Suricata, pfBlockerNG, WireGuard and Tailscale all cost nothing. The spend here reflects two years of choices for headroom and learning, not the minimum to reproduce it.

---

## Network design

<p align="center">
  <img src="assets/diagrams/network-topology.png" alt="Network topology" width="100%"/>
</p>

Devices in one segment cannot reach another unless a rule explicitly allows it.

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
| IPv6 dropped | Supporting both address families doubles the rule surface and adds silent failure modes |
| Resolver lock | DNS queries leave only through the active VPN tunnel |
| Fail closed | If both tunnels drop, traffic stops. There is no fallback to an unencrypted path |

Segment 50 is the deliberate exception. It reaches the internet directly so the firewall stays reachable when the tunnels themselves are the problem, and it carries no administrative privileges.

---

## Verification

Claims here are tested rather than assumed. Procedures in [testing-procedures.md](operations/testing-procedures.md).

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
| Observability | No metrics or dashboards. Every incident here was detected late, which is the open gap this lab most needs to close |
| One firewall | No redundant pair. A port is reserved for a future second unit |
| Switch capability | This model cannot separate management traffic onto its own segment, capping how far the design can be hardened. [Post-mortem](docs/LIMITATIONS.md) |
| Security blocking is network-wide | The intrusion detection block list applies everywhere at once, not per segment. Caused four outages. [Post-mortem](docs/LIMITATIONS.md) |
| No second factor on the firewall login | pfSense offers none natively in either edition. Risk accepted for a single-occupant network with no inbound exposure, with reasoning and reattempt criteria recorded. [Detail](docs/LIMITATIONS.md) |

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
| Wireless hardening, WPA2/WPA3 with PMF | Done, September 2026 |
| Observability: metrics, dashboards, alerting | Next. Closes the detection gap behind every incident so far |
| Configuration drift detection | Next. A restore has silently reverted security settings twice |
| Threat rules on internal segments | In detection-only first |
| Outbound filtering for smart devices | Planned |
| Managed switch replacement | Planned. Unblocks port security and the remaining hardening |
| Redundant firewall pair | Under consideration |
| CCNA exam | Scheduled, October 2026 |

</details>

---

## Background

System administrator and network engineer, 7+ years across healthcare, education and MSP environments. Active Directory, Intune, pfSense, HIPAA compliance.

This lab is where ideas get tested properly before they inform production work, and where the failures get written down.

[ajayangdembe.com](https://www.ajayangdembe.com)

---

## License

MIT. See [LICENSE](LICENSE). Use and adapt freely; keep the copyright notice and a link back if you republish substantial parts.

<div align="center">

*[Aj-Networks](https://github.com/Aj-Networks)*

</div>

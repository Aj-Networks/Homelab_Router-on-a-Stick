<div align="center">

# 🏠 Homelab - pfSense Router-on-a-Stick

📖 [Known limitations and lessons learned](docs/LIMITATIONS.md)

**A privacy-first home network built from off-the-shelf gear.**
*6 VLANs · dual VPN failover · layered kill switch · zero leaks*

![pfSense](https://img.shields.io/badge/pfSense-2.8.1-orange?logo=pfsense&logoColor=white)
![UniFi](https://img.shields.io/badge/UniFi-U7%20Lite-blue?logo=ubiquiti&logoColor=white)
![Mullvad](https://img.shields.io/badge/VPN-Mullvad%20WireGuard-yellow)
![Status](https://img.shields.io/badge/Lab-Active-brightgreen)
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)](LICENSE)
![Last commit](https://img.shields.io/github/last-commit/Aj-Networks/Homelab_Router-on-a-Stick)

</div>

> [!IMPORTANT]
> 📜 **Free to use, fork, and learn from.** If this repo helps you and you build on it or share it, a credit back is appreciated. Full terms in the [LICENSE](LICENSE).

---

## 🆕 Recent updates

| Date | Change |
|---|---|
| **2026-05-27** | Enabled **dedicated OOB management port** on Protectli Port 3 (igb2, `172.16.99.0/24`). Direct ethernet recovery path that survives any LAN/trunk misconfiguration. Enterprise pattern. |
| **2026-05-27** | Attempted native VLAN 999 + dedicated mgmt VLAN hardening. Hit GS308E v4 hardware limit ([details](docs/LIMITATIONS.md)). Closed Part 1 at 9/10 of enterprise hardening; the 1-point gap is hardware-bounded. |
| **2026-05-27** | GS308E port allocation **locked** at v1.0 (full Part 1 layout). See [switch-port-map.md](configs/switch-port-map.md). |
| **2026-05** | Replaced R6400 with **UniFi U7 Lite** (WiFi 7, VLAN-tagged SSIDs, stable AP mode) |
| **2026-05** | Added **Mac Mini M4** as 24/7 Docker host for self-hosted services |
| **2026-05** | Migrated VPN from US exits (CHI/NYC) to EU exits (Sweden/Frankfurt) |

---

## 🎯 What this is

- A single-firewall home network where **no traffic ever leaves the house outside an encrypted tunnel**
- Six VLANs, default-deny inter-VLAN, ISP sees nothing useful
- A learning lab: built to be broken, fixed, and documented

> *Not enterprise. One person learning by doing, then writing it down so it sticks.*

---

## 📊 At a glance

| | |
|---|---|
| **VLANs** | 6 (Trusted, IoT, Guest, Lab, Mgmt, Native) |
| **VPN tunnels** | 2 (Mullvad Sweden + Germany, auto-failover) |
| **Kill switch layers** | 7 (NAT, DoH/DoT block, port 53 block, RFC1918 block, IPv6 block, DNS lockdown, no WAN egress NAT) |
| **Verified zero leaks** | DNS, IP, WebRTC (ipleak.net + Mullvad Check) |
| **Total hardware cost** | ~$1,960 |

---

## 🛠 The stack

| Layer | Tool |
|---|---|
| Firewall + Router | Protectli FW6E running pfSense 2.8.1 |
| Switch | Netgear GS308E v4 (802.1Q trunking) |
| Wi-Fi | UniFi U7 Lite (WiFi 7, managed via UniFi controller) |
| VPN | Mullvad WireGuard, dual-tunnel failover |
| IDS / DNS filter | Suricata + pfBlockerNG-devel |
| Overlay / remote access | Tailscale |
| App host | Mac Mini M4 (24/7 Docker host) |
| Lab gear | Cisco Catalyst 3560 PoE-8 (WS-C3560-8PC) + Cisco 1941 ISR (CISCO1941), isolated on VLAN 40 |
| CCNA target | Hands-on prep on the Cisco gear, scheduled for the Aug 2026 exam |

---

## 🗺 Roadmap

| Item | Status |
|---|---|
| AP hardware upgrade | ✅ Done (U7 Lite, May 2026) |
| Guest VLAN tagging (real VLAN 30) | ✅ Done (May 2026, via U7 Lite trunk to GS308E) |
| Switch port allocation + labels (locked Part 1 layout) | ✅ Done (May 2026) |
| Dedicated OOB management port (Protectli Port 3) | ✅ Done (May 2026, `172.16.99.0/24`) |
| Native VLAN 999 + dedicated mgmt VLAN | ⛔ Hardware-blocked, see [LIMITATIONS.md](docs/LIMITATIONS.md) |
| AdGuard Home (network-wide ad/tracker filter) | 🔵 Under review |
| Centralized syslog server | 🔵 Exploring |

---

## 📈 Lab evolution

How this lab grew, one layer at a time. Full changelog in [`CHANGELOG.md`](CHANGELOG.md).

| Era | Milestone |
|---|---|
| **2017-2023** | Self-taught Cisco IOS on the Catalyst 3560 PoE-8 and 1941 router. VLANs, trunking, ACLs, basic routing. CCNA self-study, no public docs from this period |
| **Early 2024** | Acquired Protectli FW6E. pfSense 2.8.1, ISP modem to bridge mode, stateful firewall replaces consumer NAT. GS308E v4 added for 802.1Q trunking |
| **Mid / Late 2024** | First Mullvad WireGuard tunnel (Chicago). Policy-based outbound routing per VLAN. Kill switch via zero WAN egress NAT rules |
| **2025** | DoH/DoT blocking on ports 443 and 853. Unbound resolver. pfBlockerNG-devel for DNSBL and IP reputation. Segmentation expanded to 6 VLANs with default-deny inter-VLAN |
| **Late 2025 / Early 2026** | Second Mullvad tunnel (NYC) in a `VPN_FAILOVER` gateway group. Suricata IDS across 5 interfaces with ET Open. Full firewall rule audit and naming standardization |
| **Spring 2026** | Tailscale subnet routing for remote admin. R6400 retired, UniFi U7 Lite adopted. Mac Mini M4 added as 24/7 Docker host. VPN migrated from US to EU exits (Sweden + Frankfurt) |

---

## 🧭 Network architecture

<p align="center">
  <img src="diagrams/network-topology.png" alt="Network topology" width="100%"/>
</p>

---

## 🔒 Privacy layers (the kill switch)

Seven independent defenses. A packet has to bypass **all of them** to leak.

<table>
<tr>
<td align="center" width="33%">🛡️<br/><b>1. NAT lock</b><br/><sub>Outbound NAT bound to VPN interfaces only. No WAN egress rules exist.</sub></td>
<td align="center" width="33%">🔐<br/><b>2. Encrypted DNS block</b><br/><sub>Ports 443/853 blocked. Apps can't bypass with DoH/DoT.</sub></td>
<td align="center" width="33%">📡<br/><b>3. Plain DNS block</b><br/><sub>Port 53 to WAN blocked. No DNS escape route.</sub></td>
</tr>
<tr>
<td align="center">🚧<br/><b>4. Inter-VLAN block</b><br/><sub>Default-deny on RFC1918 (/8, /12, /16). Segmentation enforced.</sub></td>
<td align="center">🚫<br/><b>5. IPv6 block</b><br/><sub>All IPv6 dropped. No v6 leak vector.</sub></td>
<td align="center">🎯<br/><b>6. DNS over VPN</b><br/><sub>System DNS pinned to Mullvad inside the tunnel. ISP sees nothing.</sub></td>
</tr>
<tr>
<td colspan="3" align="center">
☠️ <b>7. Total kill. Both tunnels down = traffic dropped.</b> No fallback. No silent failure.
</td>
</tr>
</table>

> 🟢 **VLAN 50 is the only exception**. Intentionally outside the VPN for emergency admin access when something is broken and you need to reach the firewall directly.

---

## 🗂 VLAN & IP plan

<p align="center">
  <img src="diagrams/vlan-ip-detail.png" alt="VLAN and IP detail" width="100%"/>
</p>

**Subnet convention:** third octet matches VLAN ID. Logs read at a glance.

---

## 📚 Documentation

**Config breakdowns** (markdown, written to explain reasoning):

- [`firewall-rules.md`](configs/firewall-rules.md) · per-VLAN rule chains
- [`nat-rules.md`](configs/nat-rules.md) · the 12 outbound NAT rules
- [`vlan-assignments.md`](configs/vlan-assignments.md) · VLAN map
- [`switch-port-map.md`](configs/switch-port-map.md) · GS308E ports
- [`vpn-failover.md`](configs/vpn-failover.md) · WireGuard tunnels + gateway groups
- [`tailscale.md`](configs/tailscale.md) · overlay + ACL
- [`mac-mini/`](configs/mac-mini/) · Docker host
- [`ccna-lab/`](configs/ccna-lab/) · Cisco practice lab

**Deep dive:**

- [`LAB_TECHNICAL_GUIDE.md`](docs/LAB_TECHNICAL_GUIDE.md) · 29-section technical writeup for learners
- [`HOME_LAB_v2.pdf`](docs/HOME_LAB_v2.pdf) · full architecture writeup
- [`FIREWALL_RULES_MANUAL.pdf`](docs/FIREWALL_RULES_MANUAL.pdf) · firewall manual
- [`SWITCH_VLAN_MANUAL.pdf`](docs/SWITCH_VLAN_MANUAL.pdf) · switch + VLAN manual

---

## ✅ What I've tested

- ✅ Zero IP / DNS / WebRTC leaks ([ipleak](screenshots/ipleak.png) · [Mullvad Check](screenshots/mullvad.png))
- ✅ VPN failover (primary down → secondary promoted, no dropped traffic)
- ✅ Inter-VLAN isolation (cross-VLAN pings blocked)
- ✅ Kill switch (both tunnels down → all traffic blocked, no WAN fallback)

---

## ⚠️ Known limitations

- **Single firewall = single point of failure**. No HA pair, acceptable trade-off for home
- **Suricata in alert-only mode**. Tuning false positives before flipping to block
- **GS308E v4 hardware ceiling**. No Management VLAN feature, no SNMP, no SSH, no ACLs. Caps the network at 9/10 enterprise-grade. Full post-mortem and reattempt criteria in [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md).

Full catalogue with post-mortem write-ups: [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md)

---

## 👤 About the builder

System Administrator and Network Engineer with 7+ years across healthcare, education, and MSP environments. Skilled in Active Directory, Intune, pfSense, HIPAA compliance, and zero-trust thinking.

This homelab is where I keep hands-on networking skills sharp outside the day job, and where ideas (default-deny segmentation, verifiable zero-leak defaults, separation of firewall vs application duties) get tested before they show up in production thinking.

More projects and writeups: [ajayangdembe.com](https://www.ajayangdembe.com)

---

## 📄 License and Attribution

**MIT License.** See [LICENSE](LICENSE) for the full text.

> [!CAUTION]
> ✅ **You CAN:** use, fork, study, adapt, and build on this repo for learning, teaching, your own use, or commercial work.
>
> ❌ **You CANNOT:** strip attribution and republish this repo (or substantial parts of it) as your own original creation.
>
> 📌 **If you reference or reuse content from this repo, you MUST credit the original** and include the MIT copyright notice. Link back to https://github.com/Aj-Networks/Homelab_Router-on-a-Stick. This is legally required by the MIT License terms.

---

<div align="center">

*Built by [Aj-Networks](https://github.com/Aj-Networks) · documented so future-me remembers, in case it helps someone else who's starting out.*

</div>

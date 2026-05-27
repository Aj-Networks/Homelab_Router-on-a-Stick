<div align="center">

# 🏠 Homelab - pfSense Router-on-a-Stick

**A privacy-first home network built from off-the-shelf gear.**
*6 VLANs · dual VPN failover · layered kill switch · zero leaks*

![pfSense](https://img.shields.io/badge/pfSense-2.8.1-orange?logo=pfsense&logoColor=white)
![UniFi](https://img.shields.io/badge/UniFi-U7%20Lite-blue?logo=ubiquiti&logoColor=white)
![Mullvad](https://img.shields.io/badge/VPN-Mullvad%20WireGuard-yellow)
![Status](https://img.shields.io/badge/Lab-Active-brightgreen)
![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey)
![Last commit](https://img.shields.io/github/last-commit/Aj-Networks/Homelab_Router-on-a-Stick)

</div>

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
| **Total hardware cost** | ~$1,160 |

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

## 🆕 Recent updates

| Date | Change |
|---|---|
| **2026-05** | Replaced R6400 with **UniFi U7 Lite** (WiFi 7, VLAN-tagged SSIDs, stable AP mode) |
| **2026-05** | Added **Mac Mini M4** as 24/7 Docker host for self-hosted services |
| **2026-05** | Migrated VPN from US exits (CHI/NYC) to EU exits (Sweden/Frankfurt) |

---

## 🗺 Roadmap

| Item | Status |
|---|---|
| AP hardware upgrade | ✅ Done (U7 Lite, May 2026) |
| Guest VLAN tagging (real VLAN 30) | 🟡 In progress |
| AdGuard Home (network-wide ad/tracker filter) | 🔵 Under review |
| Centralized syslog server | 🔵 Exploring |
| Switch port hardening | ⚪ Planned |

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

**Data flow (top to bottom):**

```text
        ┌─────────┐
        │   ISP   │
        └────┬────┘
             │  WAN  (igb0)
        ┌────▼────────┐
        │   pfSense   │  routing · NAT · firewall · VPN
        │  Protectli  │  Suricata IDS · pfBlockerNG
        └────┬────────┘
             │  Trunk (igb1, 802.1Q)
        ┌────▼────────┐
        │   GS308E    │  VLAN tagging
        └─┬──┬──┬──┬──┘
          │  │  │  └── port 8 → VLAN 50 mgmt
          │  │  └───── port 6,7 → VLAN 40 lab
          │  └──────── port 4,5 → VLAN 20 IoT
          └─────────── port 2,3 → VLAN 10 trusted
                         ↓
                   ┌──────────┐
                   │ Wi-Fi AP │  UniFi U7 Lite
                   │ Mac Mini │  Docker host
                   │ Wired PC │  trusted clients
                   └──────────┘
```

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

```text
┌─ VLAN 10 ─ 10.10.10.0/24 ─ trusted ───┐
│  laptops · phones · UniFi mgmt        │  VPN
└────────────────────────────────────────┘

┌─ VLAN 20 ─ 10.10.20.0/24 ─ IoT ───────┐
│  printers · smart bulbs · cameras     │  VPN
└────────────────────────────────────────┘

┌─ VLAN 30 ─ 10.10.30.0/24 ─ guest ─────┐
│  visitor Wi-Fi · isolated from LAN    │  VPN
└────────────────────────────────────────┘

┌─ VLAN 40 ─ 10.10.40.0/24 ─ CCNA lab ──┐
│  Cisco 3560 · 1900 · console traffic  │  VPN
└────────────────────────────────────────┘

┌─ VLAN 50 ─ 10.10.50.0/24 ─ mgmt ──────┐
│  emergency admin · direct WAN egress  │  NO VPN
└────────────────────────────────────────┘
```

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
- **Guest VLAN tagging in progress**. Primary SSID up, guest SSID + trunk port reconfig pending
- **Suricata in alert-only mode**. Tuning false positives before flipping to block

---

## 👤 About the builder

System Administrator and Network Engineer with 7+ years across healthcare, education, and MSP environments. Skilled in Active Directory, Intune, pfSense, HIPAA compliance, and zero-trust thinking.

This homelab is where I keep hands-on networking skills sharp outside the day job, and where ideas (default-deny segmentation, verifiable zero-leak defaults, separation of firewall vs application duties) get tested before they show up in production thinking.

More projects and writeups: [ajayangdembe.com](https://www.ajayangdembe.com)

---

## 📜 License

Released under [Creative Commons Attribution 4.0 (CC BY 4.0)](LICENSE). Use it, fork it, build on it. Just credit the source.

---

<div align="center">

*Built by [Aj-Networks](https://github.com/Aj-Networks) · documented so future-me remembers, in case it helps someone else who's starting out.*

</div>

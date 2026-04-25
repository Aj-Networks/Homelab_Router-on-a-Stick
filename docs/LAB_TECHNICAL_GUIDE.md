# Homelab Technical Guide — Router-on-a-Stick with VLAN Segmentation, Dual-VPN Failover, and Layered Kill Switch

**A comprehensive educational walkthrough of a small-scale network lab, written for technical readers who know networking basics and want to see how the pieces fit together in practice.**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [About This Lab](#2-about-this-lab)
3. [Target Audience](#3-target-audience)
4. [Architectural Overview](#4-architectural-overview)
5. [Hardware Inventory](#5-hardware-inventory)
6. [Network Topology — Router-on-a-Stick](#6-network-topology--router-on-a-stick)
7. [VLAN Architecture — IEEE 802.1Q](#7-vlan-architecture--ieee-8021q)
8. [IP Addressing Plan](#8-ip-addressing-plan)
9. [Switching and Trunking](#9-switching-and-trunking)
10. [DHCP — Dynamic Host Configuration Protocol](#10-dhcp--dynamic-host-configuration-protocol)
11. [DNS Architecture](#11-dns-architecture)
12. [Firewall Philosophy](#12-firewall-philosophy)
13. [Per-VLAN Firewall Rules — Line by Line](#13-per-vlan-firewall-rules--line-by-line)
14. [NAT — Network Address Translation](#14-nat--network-address-translation)
15. [VPN Architecture — WireGuard with Failover](#15-vpn-architecture--wireguard-with-failover)
16. [Kill Switch — The Five Layers](#16-kill-switch--the-five-layers)
17. [DNS Filtering with pfBlockerNG](#17-dns-filtering-with-pfblockerng)
18. [Intrusion Detection — Suricata](#18-intrusion-detection--suricata)
19. [Remote Access — Tailscale](#19-remote-access--tailscale)
20. [Backup and Restore](#20-backup-and-restore)
21. [Testing and Verification](#21-testing-and-verification)
22. [Threat Model](#22-threat-model)
23. [Defense in Depth](#23-defense-in-depth)
24. [What Was Done Well](#24-what-was-done-well)
25. [Trade-offs and Limitations](#25-trade-offs-and-limitations)
26. [What We Chose Not To Do — and Why](#26-what-we-chose-not-to-do--and-why)
27. [Roadmap](#27-roadmap)
28. [Glossary](#28-glossary)
29. [References and Further Reading](#29-references-and-further-reading)

---

## 1. Executive Summary

This lab is a small but deliberately-built home network running on **pfSense 2.8.1** (an open-source firewall and routing distribution based on FreeBSD). It implements a **Router-on-a-Stick** topology, in which a single physical link carries all Virtual Local Area Network (VLAN) traffic between the firewall and a managed switch. Six VLANs separate device classes by trust level. Outbound traffic is forced through one of two **WireGuard** Virtual Private Network (VPN) tunnels to Mullvad, with automatic failover. A five-layer "kill switch" guarantees that if both tunnels go down, the firewall blocks egress instead of leaking traffic to the Internet Service Provider (ISP).

The lab also runs:

- **pfBlockerNG-devel** for Domain Name System Block List (DNSBL) and IP-level threat filtering.
- **Suricata** as a signature-based Intrusion Detection System (IDS).
- **Tailscale** for remote access via a WireGuard mesh, locked down with role-based Access Control Lists (ACLs).

Everything is documented in Markdown files alongside the live configuration, so future-you can rebuild from scratch.

---

## 2. About This Lab

### 2.1 Origin

The lab began in 2024 with a Protectli FW6E mini-PC, a fresh pfSense install, and a single Mullvad WireGuard tunnel. It evolved through 2025 as the author learned about VLANs, Network Address Translation (NAT) discipline, dual-tunnel failover, and Domain Name System (DNS) hygiene. By early 2026 it reached its current form: six VLANs, dual VPN, kill switch, IDS, DNSBL, and Tailscale.

### 2.2 Goals

| Goal | How it is achieved |
|---|---|
| Learn networking fundamentals hands-on | Build, break, fix, document |
| Achieve real (not theatrical) privacy | All client traffic egresses through Mullvad VPN; no ISP DNS fallback |
| Hard isolation between trust zones | VLANs + per-interface firewall rules + RFC 1918 inter-VLAN block |
| Prevent leaks if anything fails | Five-layer kill switch — manual NAT, DoH/DoT block, port 53 block, RFC 1918 block, IPv6 block |
| Reproducibility | Markdown docs in `/configs` describe every rule and choice |

### 2.3 Non-goals

- **Enterprise availability.** No High Availability (HA) pair, no Carrier-grade Network Address Translation (CGNAT), no Border Gateway Protocol (BGP). Single firewall, single ISP link.
- **Compliance certification.** Not designed for Payment Card Industry Data Security Standard (PCI DSS), Health Insurance Portability and Accountability Act (HIPAA), or System and Organization Controls (SOC) audits.
- **Public-facing services.** No inbound services exposed to the WAN.

---

## 3. Target Audience

This guide is written for:

- **Networking students** who understand the Open Systems Interconnection (OSI) model in theory but want to see real Layer 2 and Layer 3 boundaries enforced in a working network.
- **Self-taught system administrators** (sysadmins) considering a similar build.
- **Privacy-focused users** evaluating whether a VPN-on-router design actually prevents leaks.
- **Hiring managers and reviewers** assessing the author's hands-on networking skill.

Required background:

- IPv4 addressing and Classless Inter-Domain Routing (CIDR) notation (e.g., `10.10.10.0/24`)
- Stateful firewall basics (allow / deny / state table)
- DNS basics (resolver, recursive query, A record)
- The general idea of a VPN

Not required:

- Cisco command-line interface (CLI) experience
- pfSense familiarity (everything is shown step by step)
- Programming or scripting

---

## 4. Architectural Overview

### 4.1 Logical layers

```
+--------------------------------------------------------------+
|  Internet (ISP)                                              |
+--------------------------------------------------------------+
                |
                |  WAN  (igb0)
                v
+--------------------------------------------------------------+
|  pfSense 2.8.1  on Protectli FW6E                            |
|  ------------------------------------------                  |
|  - Stateful packet filter  (pf)                              |
|  - Manual Outbound NAT                                        |
|  - WireGuard:    VPN_CHI  (primary)                           |
|                  VPN_NYC  (failover)                          |
|  - Gateway group: VPN_FAILOVER                                |
|  - Unbound DNS resolver                                       |
|  - pfBlockerNG-devel (DNSBL + IP feeds)                       |
|  - Suricata IDS                                               |
|  - Tailscale subnet router                                    |
+--------------------------------------------------------------+
                |
                |  Trunk  (igb1, 802.1Q)
                v
+--------------------------------------------------------------+
|  Netgear GS308E v4 — managed L2 switch                        |
|  Ports 2–8 distribute access to VLANs 10, 20, 40, 50          |
+--------------------------------------------------------------+
        |          |         |         |         |
   VLAN 10    VLAN 20   VLAN 40   VLAN 50   ...
   Users     IoT       Lab       Mgmt
```

### 4.2 The four security planes

| Plane | What it does | Where it lives |
|---|---|---|
| **Layer 2 isolation** | Tags frames per VLAN so devices on different VLANs cannot share a broadcast domain | GS308E + pfSense `igb1.X` sub-interfaces |
| **Layer 3 firewalling** | Stateful per-interface rules; default deny inbound, scoped allows outbound | pfSense `pf` |
| **Egress control** | Manual outbound NAT only on VPN tunnels — no NAT on the WAN interface | pfSense NAT > Outbound |
| **Threat filtering** | DNSBL sinkholes + IPv4 deny lists + IDS pattern matching | pfBlockerNG + Suricata |

Each plane operates independently. A failure in one (e.g., a misconfigured firewall rule) does not collapse the others (the kill switch still holds because NAT and DNS lock-in are separate).

---

## 5. Hardware Inventory

| Device | Role | Notes |
|---|---|---|
| **Protectli FW6E** (Intel i7, 16 GB Random Access Memory (RAM), 6× Intel `igb` Network Interface Card (NIC)) | Firewall / Router | Runs pfSense 2.8.1. Six 1 GbE ports — only `igb0` (WAN) and `igb1` (trunk) are active; `igb2`–`igb5` are reserved for future use (Out-of-Band, Lab Direct, High Availability, Expansion). |
| **Netgear GS308E v4** | Managed L2 switch | Supports IEEE 802.1Q VLAN tagging via the proprietary "ProSafe Plus" Windows utility. 8 ports. |
| **Netgear R6400** | Wireless Access Point (AP) | Set to AP-only mode (routing disabled). Cannot do per-Service Set Identifier (SSID) VLAN tagging — see [§ 25](#25-trade-offs-and-limitations). |
| **Cisco Catalyst 3560** + **Cisco 1900** | Lab study gear | Powered down most of the time; isolated on VLAN 40 when active. |

### 5.1 Why this hardware

- **Protectli FW6E** — passive cooling, FreeBSD-friendly Intel NICs, enough horsepower for Suricata and pfBlockerNG concurrent with WireGuard at 1 Gbps.
- **GS308E** — cheapest reliable 802.1Q-capable switch at the time of build. Trade-off: no Simple Network Management Protocol (SNMP), no Secure Shell (SSH), config is binary.
- **R6400** — already on hand. Known limitation: not VLAN-aware. See § 25 and § 27 for the upgrade plan.

---

## 6. Network Topology — Router-on-a-Stick

### 6.1 What is Router-on-a-Stick?

A **Router-on-a-Stick** is a topology in which one physical interface on the router carries multiple VLANs, encoded as 802.1Q-tagged frames, instead of using one physical interface per VLAN. The switch presents a single **trunk port** to the router; the router uses **sub-interfaces** (e.g., `igb1.10`, `igb1.20`) to terminate each VLAN.

```
                       +--------+
                       |Router  |
                       |        |
                       |igb1    |   <- one physical port
                       +---+----+
                           |
                  trunk (802.1Q tagged)
                           |
                       +---+----+
                       |Switch  |
                       +-+-+-+-++
                         | | | |
                       VLAN 10 20 30 50  <- access ports
```

### 6.2 Why use it here?

| Reason | Detail |
|---|---|
| **Simpler cabling** | One Ethernet cable from firewall to switch carries all VLANs |
| **Cheaper hardware** | No need for a 6-port firewall to back 6 VLANs |
| **Pedagogically clean** | Forces you to understand 802.1Q tagging — there's no shortcut |
| **Flexibility** | Add a VLAN by editing the switch and adding `igb1.X` on the router; no rewiring |

### 6.3 Trade-offs

- The trunk link is a **single point of failure**. If the cable between firewall and switch fails, every VLAN drops.
- Throughput is shared: all VLANs combined cannot exceed the 1 Gbps physical link.
- Misconfiguration of the trunk (e.g., forgetting to tag a VLAN) silently breaks isolation.

---

## 7. VLAN Architecture — IEEE 802.1Q

### 7.1 What is a VLAN?

A **Virtual Local Area Network (VLAN)** is a Layer 2 broadcast domain that is logically separate from other VLANs even when they share the same physical infrastructure. VLANs are defined by **tags** inserted into Ethernet frames per the **IEEE 802.1Q** standard. Each VLAN has a 12-bit identifier (1 to 4094).

A frame carrying a VLAN tag looks like:

```
+----------+----------+--------+-----+----------+----------+-----+
| Dst MAC  | Src MAC  | 0x8100 | TCI | EtherType| Payload  | FCS |
| (6B)     | (6B)     | (2B)   | (2B)| (2B)     | (...)    | (4B)|
+----------+----------+--------+-----+----------+----------+-----+
                                |
                        TCI = PCP(3b) + DEI(1b) + VID(12b)
```

- **TPID `0x8100`** = Tag Protocol Identifier; tells the switch this frame is tagged
- **TCI** = Tag Control Information (priority, drop-eligible bit, VLAN ID)

A switch port can be:

- **Access** — the device on this port has no idea about VLANs; the switch tags incoming frames with the port's Port VLAN ID (PVID) and strips tags before sending out
- **Trunk** — the device on this port understands tags; the switch passes tagged frames in both directions

### 7.2 VLANs in this lab

| VLAN ID | Name | Purpose | Trust |
|---|---|---|---|
| 1 (native) | LAN_NATIVE | Trunk native, used for switch management at `10.10.1.100` | Medium |
| 10 | VLAN10_USERS | PCs, phones, Wi-Fi for trusted devices | High |
| 20 | VLAN20_IOT | Internet of Things (IoT) — printer, smart devices | Low |
| 30 | VLAN30_GUEST | Guest Wi-Fi (currently shares VLAN 10 — see § 25) | Lowest |
| 40 | VLAN40_LAB | Cisco study gear, fully isolated | Quarantine |
| 50 | VLAN50_MGMT | Management escape-hatch — direct WAN access | Privileged |

### 7.3 Why this VLAN scheme

- **Trust hierarchy** — VLANs are arranged from "trusted users" to "untrusted IoT/guest" to "isolated lab", with management as a separate privileged plane.
- **Minimum number of VLANs that does the job** — splitting further (e.g., separate VLAN per IoT category) adds operational overhead with little incremental benefit at home scale.
- **Numeric mnemonic** — the third octet of every subnet matches the VLAN ID (`10.10.20.x` ↔ VLAN 20). This makes packet-capture log review faster.

---

## 8. IP Addressing Plan

### 8.1 RFC 1918 private space

All internal subnets are drawn from **RFC 1918** ("Address Allocation for Private Internets"), specifically the `10.0.0.0/8` block.

### 8.2 Per-VLAN allocation

| VLAN | Subnet (CIDR) | Network address | Broadcast | Usable | Gateway |
|---|---|---|---|---|---|
| 1 | `10.10.1.0/24` | 10.10.1.0 | 10.10.1.255 | 10.10.1.1 – .254 | 10.10.1.1 |
| 10 | `10.10.10.0/24` | 10.10.10.0 | 10.10.10.255 | 10.10.10.1 – .254 | 10.10.10.1 |
| 20 | `10.10.20.0/24` | 10.10.20.0 | 10.10.20.255 | 10.10.20.1 – .254 | 10.10.20.1 |
| 30 | `10.10.30.0/24` | 10.10.30.0 | 10.10.30.255 | 10.10.30.1 – .254 | 10.10.30.1 |
| 40 | `10.10.40.0/24` | 10.10.40.0 | 10.10.40.255 | 10.10.40.1 – .254 | 10.10.40.1 |
| 50 | `10.10.50.0/24` | 10.10.50.0 | 10.10.50.255 | 10.10.50.1 – .254 | 10.10.50.1 |

### 8.3 Why `/24`?

- 254 usable hosts per VLAN — far more than needed today, but cheap to allocate.
- Every subnet shares an identical netmask (255.255.255.0), simplifying mental arithmetic.
- Easy migration path if a VLAN ever needs to be split or VLSM (Variable-Length Subnet Masking) is introduced later.

### 8.4 Special address — the DNSBL Virtual IP

`10.10.99.1/32` is a **Virtual IP (VIP)** mounted on the pfSense `Localhost` interface. It is **not** a VLAN gateway, not on any physical interface, and not routable from outside pfSense. It serves the pfBlockerNG block page when a domain is sinkholed. The `/32` mask says "exactly one host, no subnet."

`10.10.99.0/24` was chosen specifically because it does **not** overlap any active VLAN (1, 10, 20, 30, 40, 50) or any Tailscale-advertised route. This avoids accidental address collisions.

---

## 9. Switching and Trunking

### 9.1 Switch port map

| Port | Mode | PVID (untagged) | Tagged VLANs | Connected device |
|---|---|---|---|---|
| 1 | Trunk | 1 | 10, 20, 30, 40, 50 | pfSense `igb1` (Port 2 on FW6E) |
| 2 | Access | 10 | – | Trusted PC |
| 3 | Access | 10 | – | Netgear R6400 AP |
| 4 | Access | 20 | – | Network printer |
| 5 | Access | 20 | – | Smart device |
| 6 | Access | 40 | – | Cisco Catalyst 3560 |
| 7 | Access | 40 | – | Cisco 1900 router |
| 8 | Access | 50 | – | Management spare |

### 9.2 What "PVID" means

The **Port VLAN Identifier (PVID)** is the VLAN ID assigned to untagged frames arriving on an access port. When a PC plugs into Port 2 and sends a normal Ethernet frame, the switch stamps it with VLAN 10 before forwarding. On the way back out the access port, the tag is stripped — the PC never sees it.

### 9.3 Trunk-port behaviour

Port 1 (the trunk to pfSense) carries:

- **VLAN 1 untagged** (the "native" VLAN). Switch management frames originate here.
- **VLANs 10, 20, 30, 40, 50 tagged.**

When pfSense receives a frame on `igb1`, it inspects the 802.1Q tag and forwards to the corresponding sub-interface (`igb1.10`, `igb1.20`, etc.). Untagged frames go to the parent `igb1` interface (LAN_NATIVE).

### 9.4 Why VLAN 50 is on Port 8

Port 8 is intentionally configured as the management escape-hatch. Plugging a laptop into Port 8 places it on VLAN 50, which has direct WAN egress and full inter-VLAN reach. The intent is:

- **Routine workflow** — used for raw, non-VPN Internet access when the kill switch / VPN rules are in the way.
- **Emergency administration** — reach the firewall and any VLAN to fix a broken rule.

This is a deliberate trade-off: physical access to Port 8 = full network access. See § 25 for the limitation discussion.

---

## 10. DHCP — Dynamic Host Configuration Protocol

### 10.1 Per-VLAN DHCP scopes

pfSense runs a **DHCP server** on each VLAN sub-interface. When a device boots, it broadcasts a DHCP `DISCOVER`; pfSense responds with `OFFER`/`ACK` carrying:

- IP address from the configured pool
- Subnet mask (`/24` everywhere)
- Default gateway (the VLAN's `.1` address)
- DNS server (the VLAN's `.1` address — pfSense Unbound)
- Lease time

### 10.2 Address pools

| VLAN | DHCP range | Reserved |
|---|---|---|
| 1 | 10.10.1.10 – 10.10.1.99 | .1 (gateway), .100 (switch static) |
| 10 | 10.10.10.10 – 10.10.10.245 | .254 (R6400 static), .246–.253 future static |
| 20 | 10.10.20.10 – 10.10.20.254 | .1 |
| 30 | 10.10.30.10 – 10.10.30.254 | .1 |
| 40 | 10.10.40.10 – 10.10.40.254 | .1 |
| 50 | 10.10.50.10 – 10.10.50.254 | .1 |

### 10.3 Static mappings

| Device | MAC | IP | VLAN | Reason |
|---|---|---|---|---|
| Netgear GS308E switch | (configured) | 10.10.1.100 | 1 | Predictable management address |
| Netgear R6400 AP | (configured) | 10.10.10.254 | 10 | Predictable AP address |

Static DHCP reservations are preferred over manually-configured static IPs because they keep all address bookkeeping in one place (pfSense) and survive a device reset.

---

## 11. DNS Architecture

### 11.1 Resolution path

```
Client (any VLAN)
   |
   |  UDP/53 → VLAN gateway (e.g., 10.10.10.1)
   v
pfSense Unbound resolver
   |
   |  pfBlockerNG hook checks the queried name against DNSBL
   |  - if blocklisted: synthesize answer = 10.10.99.1 (DNSBL VIP)
   |  - if not: continue
   |
   |  Recursive query to upstream Mullvad resolvers
   |     100.64.0.1  via VPN_CHI
   |     100.64.0.2  via VPN_NYC
   v
Authoritative answer → Unbound → client
```

### 11.2 Why use Unbound (not Forwarder mode)?

**Unbound** is a recursive, validating, caching DNS resolver. It can:

- Perform full recursion (root → Top-Level Domain (TLD) → authoritative)
- Cache responses to reduce upstream queries
- Validate DNSSEC signatures (when enabled)
- Be hooked by pfBlockerNG for inline filtering

Running Unbound in resolver mode (rather than forwarder mode) means clients are not dependent on a single upstream and queries do not all go to one provider.

### 11.3 The DNS lock

| Layer | Mechanism |
|---|---|
| Outbound on each VLAN | Firewall rule: `Block UDP 53` to anywhere except the VLAN gateway |
| DoH/DoT block | Firewall rule: `Block` to `DOH_IPS` alias on TCP/UDP 443–853 |
| pfSense system DNS | Configured as Mullvad addresses only — no ISP fallback |
| Outbound NAT | Only the VPN tunnel interfaces have NAT rules — DNS queries cannot leave via WAN |

**Net effect:** there is no possible code path by which a client query reaches the ISP DNS or any non-Mullvad resolver. If a client tries to bypass with `dig @8.8.8.8` it gets blocked by the port-53 rule. If it tries DoH (DNS-over-HTTPS) on `1.1.1.1:443` it gets blocked by the DoH alias rule. If pfSense fails over to NYC, system DNS swings to the NYC Mullvad address — still encrypted, still inside the tunnel.

---

## 12. Firewall Philosophy

### 12.1 Default deny

pfSense uses **`pf`** (the OpenBSD packet filter, ported to FreeBSD). Each interface defaults to **deny-all-inbound**. Traffic is allowed only when an explicit `pass` rule matches.

### 12.2 Stateful inspection

Once a rule passes a connection, `pf` adds an entry to the **state table**. Subsequent packets in that flow are allowed automatically. This means:

- You only need to write rules for the **initiating direction**.
- Reply packets are matched by state, not by a separate rule.
- The state table is what makes the firewall fast — most packets never re-evaluate the rule list.

### 12.3 Top-down evaluation, first match wins

Rules are evaluated in order from top to bottom on the interface where traffic enters. The **first** rule whose criteria match decides the fate of the packet — `pass` or `block`. There is no "score" or "best match." Rule order matters.

### 12.4 The "RFC 1918 alias" trick

To enforce inter-VLAN isolation cheaply, the lab defines an alias `RFC1918` containing:

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

Every VLAN (except VLAN 1 native and VLAN 50) has a rule:

```
Block any from VLANXX_net to RFC1918 alias
```

This single rule blocks every possible inter-VLAN destination because every VLAN sits inside `10.0.0.0/8`. New VLANs are isolated by default — no rule edits needed when one is added.

---

## 13. Per-VLAN Firewall Rules — Line by Line

This section walks through each VLAN's rules. Each rule's purpose is explained below the table.

### 13.1 LAN — Native (10.10.1.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Block | TCP/UDP | LAN net | DOH_IPS alias | 443–853 | Block DoH / DoT |
| 2 | Block | TCP/UDP | LAN net | any | 53 | Block plain DNS to WAN |
| 3 | Pass | TCP | LAN net | MSFT_CONNECTIVITY alias | 443 | Microsoft connectivity check |
| 4 | Pass | any | LAN net | LAN net | any | Allow local |
| 5 | Pass | any | LAN net | any | any | Egress via VPN_FAILOVER |
| 6 | Block | IPv6 | LAN net | any | any | Drop IPv6 |

LAN (VLAN 1) is the trunk's native VLAN and is intentionally less restricted because the switch's management interface lives here. Cross-VLAN reach is allowed by design. Microsoft's Network Connectivity Status Indicator (NCSI) check (rule 3) is whitelisted via WAN so Windows does not show "no internet" when on VPN.

### 13.2 VLAN 10 — Users (10.10.10.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN10 net | 10.10.10.1 | 53 | DNS to pfSense resolver |
| 2 | Pass | TCP | VLAN10 net | 10.10.10.1 | 443 | pfSense WebUI (HTTPS only) |
| 3 | Pass | ICMP | VLAN10 net | 10.10.10.1 | – | Ping the gateway |
| 4 | Pass | TCP | VLAN10 net | 10.10.20.5 | 9100 | Allow printing to printer (RAW print) |
| 5 | Block | TCP/UDP | VLAN10 net | DOH_IPS alias | 443–853 | Block DoH / DoT |
| 6 | Block | TCP/UDP | VLAN10 net | any | 53 | Block plain DNS to WAN |
| 7 | Block | any | VLAN10 net | RFC1918 alias | any | Block inter-VLAN traffic |
| 8 | Pass | any | VLAN10 net | any | any | Egress via VPN_FAILOVER |
| 9 | Block | IPv6 | any | any | any | Drop IPv6 |

Rule 1–3: **scoped gateway access** — VLAN 10 hosts can resolve DNS, reach the WebUI, and ping the gateway. SSH and other admin services are not exposed from VLAN 10.
Rule 4: **explicit pinhole** for printing — TCP 9100 is the standard RAW Printing protocol port.
Rules 5–7: kill-switch and isolation enforcement.
Rule 8: the **default egress** through the VPN failover gateway group.
Rule 9: IPv6 dropped to eliminate dual-stack leak vectors.

### 13.3 VLAN 20 — IoT (10.10.20.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN20 net | 10.10.20.1 | 53 | DNS only |
| 2 | Block | TCP/UDP | VLAN20 net | DOH_IPS alias | 443–853 | Block DoH / DoT |
| 3 | Block | TCP/UDP | VLAN20 net | any | 53 | Block plain DNS to WAN |
| 4 | Block | any | VLAN20 net | RFC1918 alias | any | Block inter-VLAN |
| 5 | Pass | any | VLAN20 net | any | any | Egress via VPN_FAILOVER |
| 6 | Block | IPv6 | any | any | any | Drop IPv6 |

IoT devices are the **lowest-trust** category. They get DNS to the gateway and outbound Internet via the VPN, nothing else. They cannot reach the WebUI or any other VLAN.

### 13.4 VLAN 30 — Guest (10.10.30.0/24)

Same shape as VLAN 20 — DNS only to gateway, no inter-VLAN, egress via VPN. Identical rule structure because guest devices are treated as IoT-equivalent: Internet access only.

### 13.5 VLAN 40 — Lab (10.10.40.0/24)

Same shape as VLAN 20. Lab gear is treated as untrusted by default. Excluded from Suricata only when actively generating Cisco protocol noise (Cisco Discovery Protocol (CDP), Spanning Tree Protocol (STP), Dynamic Trunking Protocol (DTP)) — currently runs in alert-only mode awaiting traffic-driven SID tuning.

### 13.6 VLAN 50 — Management (10.10.50.0/24)

| # | Action | Protocol | Source | Destination | Port | Notes |
|---|---|---|---|---|---|---|
| 1 | Pass | UDP | VLAN50 net | 10.10.50.1 | 53 | DNS |
| 2 | Pass | TCP | VLAN50 net | 10.10.50.1 | 443 | WebUI |
| 3 | Pass | TCP | VLAN50 net | 10.10.50.1 | 22 | SSH |
| 4 | Pass | ICMP | VLAN50 net | 10.10.50.1 | – | Ping |
| 5 | Block | TCP/UDP | VLAN50 net | any | 53 | Block plain DNS to WAN |
| 6 | Block | TCP/UDP | VLAN50 net | DOH_IPS alias | 443–853 | Block DoH / DoT |
| 7 | Pass | any | VLAN50 net | any | any | Direct egress + full inter-VLAN |
| 8 | Block | IPv6 | VLAN50 net | any | any | Drop IPv6 |

Rule 7 is the **escape hatch**. VLAN 50 bypasses both the VPN and the inter-VLAN isolation. This is intentional: the management plane needs to reach the firewall and other VLANs when something is broken. It is the lab's break-glass network.

---

## 14. NAT — Network Address Translation

### 14.1 What NAT does

**Network Address Translation (NAT)** rewrites IP addresses (and sometimes ports) as a packet crosses a network boundary. In a home network, **Outbound NAT** (a.k.a. Source NAT or "masquerade") rewrites the *source* address of outgoing packets from the internal address (e.g., `10.10.10.50`) to the public-facing address of the external interface, so the response can be routed back.

### 14.2 Manual mode is the foundation of the kill switch

pfSense supports four Outbound NAT modes:

| Mode | Behaviour |
|---|---|
| Automatic | pfSense generates NAT rules dynamically for any RFC 1918 → WAN flow |
| Hybrid | Automatic + your manual rules layered on top |
| **Manual** | Only the rules you write are used; nothing is auto-generated |
| Disabled | No NAT at all |

This lab uses **Manual** mode. There are zero NAT rules on the WAN interface. Therefore, **even if a packet somehow reaches `igb0`, there is no NAT entry to translate it, and the ISP-facing source IP will be wrong, so reply traffic cannot return.** The flow is functionally dead.

### 14.3 The 12 outbound rules

| # | Interface | Source | Translation |
|---|---|---|---|
| 1 | VPN_CHI | 10.10.1.0/24 | VPN_CHI address |
| 2 | VPN_CHI | 10.10.10.0/24 | VPN_CHI address |
| 3 | VPN_CHI | 10.10.20.0/24 | VPN_CHI address |
| 4 | VPN_CHI | 10.10.30.0/24 | VPN_CHI address |
| 5 | VPN_CHI | 10.10.40.0/24 | VPN_CHI address |
| 6 | VPN_CHI | 10.10.50.0/24 | VPN_CHI address |
| 7 | VPN_NYC | 10.10.1.0/24 | VPN_NYC address |
| 8 | VPN_NYC | 10.10.10.0/24 | VPN_NYC address |
| 9 | VPN_NYC | 10.10.20.0/24 | VPN_NYC address |
| 10 | VPN_NYC | 10.10.30.0/24 | VPN_NYC address |
| 11 | VPN_NYC | 10.10.40.0/24 | VPN_NYC address |
| 12 | VPN_NYC | 10.10.50.0/24 | VPN_NYC address |

Two tunnels × six subnets = 12 rules. Each subnet has a NAT rule on **both** tunnels so failover is seamless.

The VLAN 50 NAT rules (6 and 12) never actually match in practice because VLAN 50 routes through WAN directly per its firewall rules. They exist so the rule list stays symmetric across tunnels.

---

## 15. VPN Architecture — WireGuard with Failover

### 15.1 What is WireGuard?

**WireGuard** is a modern VPN protocol notable for:

- **Small codebase** (~4,000 lines vs. OpenVPN's hundreds of thousands)
- **In-kernel** Linux/BSD implementation → high throughput, low latency
- **Cryptokey routing** — peer identity is the public key; no Public Key Infrastructure (PKI) ceremony
- **Stateless / no session negotiation** — clients re-establish silently after reboot

Each WireGuard tunnel is a User Datagram Protocol (UDP) flow between two endpoints, encrypted with the **Noise Protocol Framework** primitives (Curve25519, ChaCha20, Poly1305, BLAKE2s).

### 15.2 The two tunnels

| Tunnel | Provider | Exit | Local UDP port | Role |
|---|---|---|---|---|
| **VPN_CHI** | Mullvad | Chicago, IL | 51820 | Primary (Tier 1) |
| **VPN_NYC** | Mullvad | New York City, NY | 51821 | Failover (Tier 2) |

Each tunnel has its own pfSense interface (`tun_wgX`) and gateway (`GW_VPN_CHI`, `GW_VPN_NYC`).

### 15.3 The gateway group — `VPN_FAILOVER`

A **gateway group** is a pfSense construct that bundles multiple gateways with a tier ordering and a triggering policy.

| Member | Tier | Behaviour |
|---|---|---|
| `GW_VPN_CHI` | 1 | Active when up |
| `GW_VPN_NYC` | 2 | Promoted to active if Tier 1 is down |

Gateway health is monitored by pfSense via Internet Control Message Protocol (ICMP) probes (`apinger` / `dpinger` daemon). If three consecutive probes fail to reach the Mullvad endpoint, the tunnel is marked down and Tier 2 is promoted within ~30 seconds.

### 15.4 Why "VPN_FAILOVER" is referenced everywhere

Every per-VLAN egress rule (Rule 8 on VLAN 10, Rule 7 on VLAN 50, etc.) sets the gateway to **VPN_FAILOVER**, not to a specific tunnel. This way:

- Failover is automatic — no rule edits, no state flush
- The gateway abstraction insulates the rule set from tunnel changes (e.g., if you swap NYC for Toronto, only the gateway group changes)

### 15.5 DNS through the tunnel

| DNS server | Reached via |
|---|---|
| 100.64.0.1 | VPN_CHI |
| 100.64.0.2 | VPN_NYC |

These are Mullvad's internal CGNAT-range DNS resolvers, reachable **only** through the corresponding tunnel. pfSense's system DNS configuration lists both. Unbound forwards out via these. There is no fallback to ISP DNS, public DNS, or any non-Mullvad provider.

---

## 16. Kill Switch — The Five Layers

A "kill switch" is the property that **if the VPN drops, traffic is blocked, not leaked.** This lab implements it as five independent layers.

### 16.1 Layer 1 — Manual Outbound NAT with no WAN rules

Already covered in § 14. Without a WAN NAT rule, packets cannot leave the WAN interface with a usable source address. **First and most important** layer.

### 16.2 Layer 2 — DoH/DoT block

A pfSense alias `DOH_IPS` lists known DNS-over-HTTPS (DoH) and DNS-over-TLS (DoT) provider IPs. Each VLAN has a rule:

```
Block TCP/UDP from VLANXX_net to DOH_IPS:443-853
```

This prevents malware (or a misconfigured browser) from bypassing the standard DNS path with encrypted DNS. Without this layer, a leaked DNS query on port 443 to Cloudflare's `1.1.1.1` would tell the ISP nothing — but it would also bypass pfBlockerNG.

### 16.3 Layer 3 — Plain DNS block (port 53)

```
Block TCP/UDP from VLANXX_net to any on port 53
```

(Above the egress rule.) Forces all DNS to the local Unbound resolver. A `dig @8.8.8.8` from any VLAN times out.

### 16.4 Layer 4 — RFC 1918 inter-VLAN block

Prevents any VLAN from reaching another VLAN's address space. Without this, a compromised IoT device could pivot to attack the user VLAN.

### 16.5 Layer 5 — IPv6 block

```
Block IPv6 from any to any
```

IPv6 is currently disabled lab-wide. The reason: a misconfigured IPv6 stack can leak around an IPv4 VPN tunnel because most VPN providers tunnel IPv4 only. Blocking IPv6 outright eliminates that vector. Future plan: re-enable with full IPv6 firewall parity once the IPv6 NAT and pfBlockerNG IPv6 feeds are in place.

### 16.6 What "leak" looks like

| Leak vector | Mitigated by |
|---|---|
| WAN routing if VPN tunnel fails | Layer 1 (manual NAT) |
| Browser-issued DoH bypass | Layer 2 (DoH/DoT block) |
| Application using `8.8.8.8` directly | Layer 3 (port 53 block) |
| Compromised host pivoting | Layer 4 (RFC 1918 block) |
| IPv6 leak around IPv4 tunnel | Layer 5 (IPv6 block) |

---

## 17. DNS Filtering with pfBlockerNG

### 17.1 What pfBlockerNG-devel is

**pfBlockerNG-devel** is a pfSense package that provides:

- **DNSBL** — DNS Block List. Hooks into Unbound; resolves blocklisted domains to a sinkhole VIP.
- **IP feed blocking** — Aliases populated from threat-intelligence feeds, used in firewall block rules.
- **GeoIP** — Block by country code (currently disabled here; outbound is already VPN-tunneled).

### 17.2 The DNSBL VIP

`10.10.99.1/32` on the `Localhost` interface, as described in § 8.4. When a DNSBL match occurs, Unbound returns this address. The pfBlockerNG block-page web server listens on this VIP for HTTP/HTTPS and renders an explanation page.

### 17.3 Active DNSBL groups

| Group | Action | Purpose |
|---|---|---|
| `ADs_Basic` | Unbound | Core ad/tracker blocking (StevenBlack and similar) |
| `ADs` | Unbound | Extended ad/tracker coverage |
| `Firebog_Suspicious` | Unbound | Curated suspicious-domain lists |
| `Phishing` | Unbound | Phishing-Army feed |
| `BBcan177` | Unbound | Curated threat-domain lists |
| `Malicious` | Unbound | Disconnect.me Malvertising/Malware, MVPS, SomeoneWhoCares |

All groups update daily at 03:00 via the package's CRON.

### 17.4 Active IPv4 deny groups

| Group | Action | Purpose |
|---|---|---|
| PRI1_Spamhaus | Deny Both | Spamhaus DROP and EDROP — hijacked / unallocated netblocks |
| PRI1_Botnet | Deny Both | Feodo Tracker — known botnet command-and-control IPs |
| PRI1_ET | Deny Both | Emerging Threats Compromised Hosts |

These populate aliases used in firewall block rules. Outbound to a Spamhaus DROP IP, for example, never leaves the firewall.

### 17.5 The auto-generated permit rules

When DNSBL is enabled and the WebServer/VIP mode is set, pfBlockerNG creates floating firewall rules that allow client VLANs to reach the VIP on TCP 80/443 and ICMP. Without these rules, the block page would not render. The lab restricts this to VLAN10, 20, 30, and 50 — VLAN 40 is excluded because it does not need a block page when isolated.

---

## 18. Intrusion Detection — Suricata

### 18.1 What Suricata does

**Suricata** is an open-source signature-based IDS. It inspects packet streams against a rule set (Emerging Threats Open + Snort Subscriber + custom) and raises **alerts** when a pattern matches a known attack signature.

It runs in two modes:

| Mode | Behaviour |
|---|---|
| Alert-only | Logs the match; takes no action |
| Inline (`Block Offenders`) | Alerts + dynamically blocks the source IP for a configurable hold-down period |

### 18.2 Where it runs

| Interface | Mode | Notes |
|---|---|---|
| WAN | Alert-only (typical) | Visibility on inbound probes |
| VLAN10_USERS | Alert-only | User traffic |
| VLAN20_IOT | Alert-only | IoT traffic |
| VLAN30_GUEST | Alert-only | Guest traffic |
| VLAN40_LAB | Alert-only (new — observation phase) | Suppressions deferred until lab gear is live |
| VLAN50_MGMT | Alert-only | Management traffic |

### 18.3 Why alert-only?

For a homelab, full inline blocking is high-risk: a single noisy false-positive can lock you out of your own network. Alert-only gives visibility without operational hazard. Block mode is left as a future upgrade once tuning is complete.

### 18.4 Rule sources

- **Emerging Threats Open (ET Open)** — free, broad coverage
- **Snort Subscriber Ruleset** (Registered tier) — community rules
- **pfSense GPLv2 Community Rules** — bundled with the package

Categories enabled: `policy`, `web-attacks`, `current-events`, `malware`, `command-and-control`, `attack-response`, etc. The full set is too long to list inline; per-interface configuration lives in the pfSense Suricata UI.

---

## 19. Remote Access — Tailscale

### 19.1 What Tailscale is

**Tailscale** is a managed mesh VPN built on top of WireGuard. Each device runs a Tailscale client; the **coordination server** (run by Tailscale, Inc.) handles key exchange and peer discovery. Once peers are introduced, traffic flows directly device-to-device (or via DERP relay if Network Address Translation traversal fails).

Each device gets an address in the **Carrier-Grade NAT (CGNAT)** range `100.64.0.0/10` (specifically `100.64.0.0` through `100.127.255.255`). These addresses do not collide with RFC 1918 home networks.

### 19.2 Subnet routing

A Tailscale client running on the firewall can advertise **subnet routes** — physical LAN ranges that the rest of the tailnet should reach via this device. For this lab, only **`10.10.10.0/24`** is currently approved.

| Subnet | Advertised | Approved | Reason |
|---|---|---|---|
| 10.10.1.0/24 | yes | **no** | Native, no devices |
| 10.10.10.0/24 | yes | **yes** | User VLAN — only one needed remotely |
| 10.10.20.0/24 | yes | **no** | IoT, not needed |
| 10.10.30.0/24 | yes | **no** | Guest, not needed |
| 10.10.40.0/24 | yes | **no** | Lab, isolated by design |
| 10.10.50.0/24 | yes | **no** | Mgmt is physical-only |

### 19.3 Access Control Lists

Tailscale ACLs are JSON documents stored in the admin console. The lab uses:

```json
{
  "tagOwners":  { "tag:home": ["autogroup:admin"] },
  "acls":       [
    { "action": "accept",
      "src":    ["tag:home", "autogroup:admin"],
      "dst":    ["10.10.10.0/24:*"]
    }
  ],
  "ssh":        []
}
```

| Element | Meaning |
|---|---|
| `tagOwners` | Defines `tag:home` and says only admins may assign it |
| ACL `src` | Allowed source identities — devices tagged `tag:home` and any device of the admin user |
| ACL `dst` | Allowed destination — only the user VLAN, all ports |
| Implicit | Anything not explicitly accepted is **denied** |

### 19.4 Why one tag?

Single-user lab → single tag. If a second tier is needed later (e.g., a friend's device with read-only access), introduce `tag:guest` with a stricter destination set. Don't widen `tag:home`.

---

## 20. Backup and Restore

### 20.1 What gets backed up

| Source | Format | Storage |
|---|---|---|
| pfSense full config | XML, optionally encrypted | Local + offsite |
| GS308E switch | Binary `.cfg` from "ProSafe Plus" utility | Local, encrypted with `age` |
| WireGuard private keys | Text | Password manager only |
| Tailscale auth keys | Text | Password manager only |
| Mullvad account number | Text | Password manager only |

### 20.2 The 3-2-1 principle

- **3** copies of every backup
- **2** different media types
- **1** copy offsite

The lab follows this approximately: pfSense XML on the local box, on an external drive, and in encrypted cloud storage. Switch config is committed to the repository as `.age`-encrypted blobs.

### 20.3 Encryption tools used

- **age** — modern file encryption, simple key model. Replaces GPG for "encrypt this one file" workflows.
- **AutoConfigBackup** — pfSense package; encrypts XML before uploading to Netgate's free hosted tier.

### 20.4 Schedule

| When | What |
|---|---|
| Before any pfSense change | Manual XML export |
| Before any switch VLAN change | Manual `.cfg` export + screenshots |
| Monthly | Rotate offsite copy, prune older than 6 months |
| Quarterly | **Test restore** to a pfSense VM. A backup that has never been restored is theatre. |
| On key rotation | Password manager update + fresh XML export |

---

## 21. Testing and Verification

The lab maintains 10 documented, repeatable tests. Each is run after any change in scope.

| # | Test | What it proves |
|---|---|---|
| 1 | VLAN isolation matrix | Inter-VLAN block rules hold |
| 2 | DNSBL verification | Sinkhole resolves to `10.10.99.1` |
| 3 | DoH/DoT block | Encrypted DNS bypass is blocked |
| 4 | Plain DNS leak | External resolvers (`8.8.8.8`, `1.1.1.1`) time out |
| 5 | VPN kill switch drill | Both tunnels down → zero egress |
| 6 | VPN failover drill | CHI down → NYC promotes within ~30 s |
| 7 | IPv6 block | `curl -6` fails externally |
| 8 | Tailscale reach | Only approved subnets reachable from tailnet |
| 9 | Suricata health | Alert counter advances during browsing |
| 10 | External leak panel | ipleak.net + Mullvad Check + browserleaks all green |

Cadence:

| When | Tests |
|---|---|
| After firewall/NAT change | 1, 3, 4 |
| After VPN change | 5, 6, 10 |
| After pfBlockerNG update | 2 |
| After Tailscale change | 8 |
| Monthly (no changes) | 1, 2, 5, 10 |
| Before / after firmware upgrade | All |

---

## 22. Threat Model

### 22.1 In scope

| Threat | Mitigation |
|---|---|
| ISP traffic logging or selling | All client traffic egresses via Mullvad VPN |
| DNS query metadata leaking to ISP | DNS locked to Mullvad through tunnels; no fallback |
| IoT device compromise pivoting laterally | RFC 1918 inter-VLAN block |
| Browser using DoH to bypass DNS controls | DoH/DoT IP alias block |
| WireGuard tunnel failure leaking traffic | 5-layer kill switch |
| Public ad/tracker / phishing / malware DNS exfil | pfBlockerNG DNSBL |
| Known-bad outbound IP destinations | pfBlockerNG IPv4 deny groups |
| Inbound network probes | pfSense default-deny + Suricata alerts |
| Lost device on tailnet | Tailscale ACL limits blast radius to VLAN 10 |
| Local config loss | XML/CFG backups, encrypted, offsite |

### 22.2 Out of scope

| Threat | Reason |
|---|---|
| Targeted nation-state attacker | Outside realistic homelab scope |
| Physical theft of FW6E | Mitigation requires full-disk encryption + tamper detection |
| Cold-boot RAM attacks | Outside scope |
| Side-channel attacks on Mullvad provider | Trusting Mullvad is a deliberate choice |
| Supply-chain compromise of pfSense or WireGuard | Mitigated only by upstream community |

---

## 23. Defense in Depth

The same packet, hostile or benign, must traverse multiple independent controls:

```
[Client] → 802.1Q tag → Switch ACL → pfSense ingress firewall →
        → DNS resolver (Unbound + pfBlockerNG hook) →
        → Outbound firewall rules → RFC 1918 block →
        → Manual outbound NAT (VPN only) →
        → IDS inspection (Suricata) →
        → WireGuard encryption → Internet
```

A single misconfiguration cannot defeat the chain because each control is independent:

- A bad firewall rule does **not** break NAT discipline.
- A failed VPN tunnel does **not** allow plain-DNS leak (port 53 still blocked).
- A pfBlockerNG outage does **not** allow inter-VLAN traffic (RFC 1918 rule still holds).

This is the core property of **defense in depth**: redundancy across orthogonal controls.

---

## 24. What Was Done Well

| Area | Why it stands out |
|---|---|
| **Manual Outbound NAT** | The single highest-leverage choice in the build. One mode toggle creates a hard kill switch. |
| **Subnet ID = VLAN ID convention** | Logs are readable at a glance. `10.10.20.5` is unambiguously a VLAN 20 host. |
| **RFC 1918 alias for inter-VLAN block** | One rule covers every present and future VLAN. New VLANs are isolated by default. |
| **Gateway group abstraction** | Failover is automatic and transparent to firewall rules. |
| **DNS lock-in across all five layers** | Plain DNS, DoH, DoT, IPv6 — every encrypted-DNS bypass path is closed. |
| **Markdown documentation per concern** | Each `configs/*.md` describes both the *what* and the *why*. |
| **Repeatable test suite** | 10 tests with cadence and pass criteria. Verification is not a vibe. |
| **Default-deny Tailscale ACL** | Even with tagged devices, the destination set is minimal and explicit. |
| **DNSBL VIP on a non-overlapping range** | Block page works without colliding with any real subnet. |

---

## 25. Trade-offs and Limitations

### 25.1 Guest Wi-Fi shares VLAN 10

The Netgear R6400 cannot do per-SSID 802.1Q tagging. Guests technically land on VLAN 10 with AP-level client isolation enabled. AP isolation prevents guest-to-guest direct traffic but does **not** prevent a guest device on the same Layer-2 broadcast domain as the user PCs from sniffing or attempting Address Resolution Protocol (ARP) spoofing. **This is the largest residual gap.**

Fix path: replace the R6400 with a VLAN-aware AP (UniFi U6-Lite, TP-Link EAP245, or similar).

### 25.2 No Wide Area Network (WAN) High Availability

Single ISP link, single firewall. A WAN outage takes everything down. No Border Gateway Protocol multi-homing, no Common Address Redundancy Protocol (CARP) pair. Acceptable at homelab scale; explicit in the goal statement.

### 25.3 IPv6 disabled

Eliminates a leak vector but defers the IPv6 deployment problem. When the ISP eventually deprecates IPv4 or an IPv6-only service becomes interesting, this needs revisiting.

### 25.4 Single Point of Failure: the trunk link

The Cat6 cable between FW6E `igb1` and switch Port 1 carries every VLAN. Cable failure or port failure drops the entire LAN. Acceptable trade-off for the simplicity of Router-on-a-Stick.

### 25.5 Switch is consumer-grade

The GS308E:
- Cannot be managed by SSH or SNMP, only by a Windows-only utility
- Does not have port security / MAC binding
- Does not support 802.1X authentication
- Has no syslog export

For a homelab the trade-off is acceptable, but every "switch-side" hardening recommendation (port security, dot1x, etc.) is blocked on a hardware swap.

### 25.6 No 2FA on the WebUI

A single admin password is the only barrier. Mitigated by:
- WebUI not exposed to WAN
- Reachable only from VLAN 50 or VLAN 10 (which is itself behind tag-based Tailscale ACL)

Trade-off accepted because pfSense has no native Time-based One-Time Password (TOTP) support; native 2FA would require a Remote Authentication Dial-In User Service (RADIUS) server. The single-user homelab scope does not justify standing up RADIUS.

### 25.7 VLAN 50 is "physical-presence" security

Anyone who plugs a laptop into Switch Port 8 has full network access. There is no port security, MAC binding, 802.1X, or alerting on link-up. This is by design — VLAN 50 is the break-glass network. Mitigated by the building's physical access controls, not by network-side controls.

### 25.8 Suricata in alert-only mode

No automatic blocking. Alerts are useful for forensics but do not stop an attack in progress. Trade-off: avoids self-induced lockouts from false positives. Future: tune categories enough to enable inline blocking on WAN ingress at minimum.

### 25.9 No centralized logging

Logs (firewall, Suricata, pfBlockerNG) live on the FW6E box. A reboot, disk corruption, or ring-buffer rollover loses them. On the roadmap.

### 25.10 No Uninterruptible Power Supply (UPS)

A brownout drops the VPN tunnels mid-session. The kill switch should hold during reboot, but it has not been verified end-to-end through a real power loss. Add a 1500 VA UPS — known gap.

---

## 26. What We Chose Not To Do — and Why

| Decision | Reason |
|---|---|
| **No 2FA on WebUI** | Pure homelab, single user, WebUI not on WAN, RADIUS overhead not justified |
| **No GeoIP block** | Outbound is already tunneled via Mullvad — additional GeoIP filter is duplicative |
| **No VLAN-per-IoT-category** | Two overlapping IoT VLANs increase rule sprawl with marginal isolation benefit |
| **No Multi-Hop Mullvad** | Latency cost ≥ marginal privacy benefit for a homelab |
| **No public DNS forwarder** | Defeats DNS lock-in |
| **No SSH from WAN** | Increases attack surface; Tailscale provides remote shell access if ever needed |
| **No auto-block (Suricata inline)** | False-positive lockout risk too high without further tuning |
| **No HA pfSense pair** | Cost, complexity, no operational requirement |

---

## 27. Roadmap

| Item | Status | Notes |
|---|---|---|
| Tailscale remote access | Done | Subnet locked to VLAN 10, tag-based ACL, default-deny |
| Suricata IDS | Done | Live on all client VLANs; VLAN 40 alert-only awaiting traffic-driven tuning |
| pfBlockerNG DNSBL + IPv4 feeds | Done | 6 DNSBL groups, 3 IPv4 deny groups, daily updates |
| Backup procedure | Done | XML + age-encrypted CFG; quarterly restore drill on schedule |
| Repeatable test procedures | Done | 10 tests, cadence defined |
| AP hardware upgrade | On hold | Closes biggest residual gap (guest VLAN) |
| Centralized logging | Exploring | Syslog → Loki/Grafana or ELK |
| Switch port hardening | Planned | Move unused ports to a dead VLAN |
| WireGuard key rotation schedule | Planned | 90/180-day rotation cadence |
| Hardware Bill of Materials | Planned | Serials, MACs, firmware versions, warranty |
| WAN hardening checklist | Planned | Document every WAN-facing service posture |
| IPv6 enablement plan | Planned | Re-enable with full firewall and DNSBL parity |
| UPS deployment | Planned | 1500 VA, ~30 min runtime |
| Architecture Decision Records | Planned | Capture rationale for each major choice |
| Runbooks | Planned | Failover drill, restore, upgrade |

---

## 28. Glossary

| Term | Expansion / definition |
|---|---|
| **ACL** | Access Control List |
| **AP** | (Wireless) Access Point |
| **ARP** | Address Resolution Protocol |
| **BGP** | Border Gateway Protocol |
| **CARP** | Common Address Redundancy Protocol |
| **CDP** | Cisco Discovery Protocol |
| **CGNAT** | Carrier-Grade Network Address Translation |
| **CIDR** | Classless Inter-Domain Routing |
| **CRON** | Time-based job scheduler |
| **DERP** | Designated Encrypted Relay for Packets (Tailscale relay) |
| **DHCP** | Dynamic Host Configuration Protocol |
| **DNS** | Domain Name System |
| **DNSBL** | Domain Name System Block List |
| **DNSSEC** | DNS Security Extensions |
| **DoH** | DNS-over-HTTPS |
| **DoT** | DNS-over-TLS |
| **DROP** | Don't Route Or Peer (Spamhaus list) |
| **DTP** | Dynamic Trunking Protocol |
| **EDROP** | Extended DROP (Spamhaus list) |
| **ET** | Emerging Threats |
| **FCS** | Frame Check Sequence |
| **HA** | High Availability |
| **HSRP** | Hot Standby Router Protocol |
| **HTTP / HTTPS** | Hypertext Transfer Protocol / HTTP Secure |
| **ICMP** | Internet Control Message Protocol |
| **IDS** | Intrusion Detection System |
| **IoT** | Internet of Things |
| **IP** | Internet Protocol |
| **ISP** | Internet Service Provider |
| **L2 / L3** | Layer 2 / Layer 3 (OSI model) |
| **MAC** | Media Access Control |
| **NAT** | Network Address Translation |
| **NCSI** | Network Connectivity Status Indicator (Microsoft) |
| **NIC** | Network Interface Card |
| **OSI** | Open Systems Interconnection |
| **pf** | OpenBSD packet filter |
| **PKI** | Public Key Infrastructure |
| **PVID** | Port VLAN Identifier |
| **RADIUS** | Remote Authentication Dial-In User Service |
| **RAM** | Random Access Memory |
| **RFC** | Request For Comments |
| **SID** | Signature Identifier (Suricata/Snort rule ID) |
| **SNMP** | Simple Network Management Protocol |
| **SOC** | System and Organization Controls |
| **SSH** | Secure Shell |
| **SSID** | Service Set Identifier |
| **STP** | Spanning Tree Protocol |
| **TCI** | Tag Control Information (802.1Q) |
| **TCP** | Transmission Control Protocol |
| **TLS** | Transport Layer Security |
| **TOTP** | Time-based One-Time Password |
| **TPID** | Tag Protocol Identifier (802.1Q, `0x8100`) |
| **UDP** | User Datagram Protocol |
| **UPS** | Uninterruptible Power Supply |
| **VID** | VLAN Identifier |
| **VIP** | Virtual IP |
| **VLAN** | Virtual Local Area Network |
| **VLSM** | Variable-Length Subnet Masking |
| **VPN** | Virtual Private Network |
| **WAN** | Wide Area Network |

---

## 29. References and Further Reading

### Standards
- **IEEE 802.1Q-2018** — Bridges and Bridged Networks (VLAN tagging)
- **RFC 1918** — Address Allocation for Private Internets
- **RFC 1631 / 3022** — Network Address Translation
- **RFC 4291** — IP Version 6 Addressing Architecture
- **RFC 8484** — DNS Queries over HTTPS (DoH)
- **RFC 7858** — DNS over Transport Layer Security (DoT)

### Software
- pfSense — https://www.pfsense.org/
- WireGuard — https://www.wireguard.com/
- Suricata — https://suricata.io/
- pfBlockerNG-devel — https://github.com/pfsense/FreeBSD-ports/tree/devel/security/pfSense-pkg-pfBlockerNG-devel
- Unbound — https://www.nlnetlabs.nl/projects/unbound/
- Tailscale — https://tailscale.com/

### Threat-intelligence feeds used
- Spamhaus DROP / EDROP — https://www.spamhaus.org/drop/
- Feodo Tracker (abuse.ch) — https://feodotracker.abuse.ch/
- URLhaus (abuse.ch) — https://urlhaus.abuse.ch/
- Phishing Army — https://phishing.army/
- The Firebog — https://firebog.net/
- StevenBlack hosts — https://github.com/StevenBlack/hosts
- Disconnect.me — https://disconnect.me/

### Companion repository documentation
- [`README.md`](../README.md) — high-level project summary
- [`CHANGELOG.md`](../CHANGELOG.md) — phase-by-phase history
- [`configs/firewall-rules.md`](../configs/firewall-rules.md) — per-VLAN rule chains
- [`configs/nat-rules.md`](../configs/nat-rules.md) — manual outbound NAT rules
- [`configs/vlan-assignments.md`](../configs/vlan-assignments.md) — interface and subnet map
- [`configs/switch-port-map.md`](../configs/switch-port-map.md) — GS308E port-VLAN mapping
- [`configs/vpn-failover.md`](../configs/vpn-failover.md) — WireGuard tunnel and gateway-group config
- [`configs/pfblockerng.md`](../configs/pfblockerng.md) — DNSBL, IPv4, and update schedule
- [`configs/tailscale.md`](../configs/tailscale.md) — Tailscale routes, ACL, and tag policy
- [`configs/backup-procedure.md`](../configs/backup-procedure.md) — backup and restore
- [`configs/testing-procedures.md`](../configs/testing-procedures.md) — 10-test verification suite

---

*End of guide.*

# Changelog

All notable changes to this project are documented here.

---

## [Phase 3], In Progress

### Done
- Management VLAN stripped of administrative privileges (2026-07-31). That VLAN is broadcast as a Wi-Fi SSID providing no-VPN internet, and it carried pass rules for the pfSense WebUI, SSH, and an unrestricted catch-all granting reach into every other VLAN. Anyone holding that Wi-Fi password therefore had the firewall admin interface, a shell, and every device on the network, from beyond the building. The wired escape hatch on switch port 8 required physical presence; the SSID did not, and the firewall matches on source subnet rather than on how a client associated, so the two cannot be told apart. Removed both pass rules and inserted a block to the RFC1918 alias directly above the catch-all, positioned below the DNS and ICMP rules so the segment still resolves names and pings its gateway. The alias covers `172.16.0.0/12`, which is what keeps the segment away from the out-of-band management interface. Administrative access is now the trusted VLAN plus two physically-gated paths, out-of-band on Protectli Port 3 and LAN on Port 2; out-of-band reachability was verified before any rule was removed. The SSID keeps every capability actually used: a VPN-independent baseline for diagnosis, and an escape route when a VPN exit is blocked by a captcha or reputation filter. Generalised in [`network/firewall-rules.md`](network/firewall-rules.md): a privilege reachable over Wi-Fi is a privilege granted to anyone within radio range.
- Orphaned outbound NAT rules found and repaired after the tunnel replacement (2026-07-31). Deleting a VPN interface does not delete the outbound NAT rules bound to it; they persist with an empty Interface column and a raw interface-IP reference, and still look plausible in the UI. Only two of six subnets had been rebuilt against the new tunnels, so three VLANs had no NAT and no internet for an unknown period, while the management VLAN masked the problem by continuing to work through its WAN rule. Surfaced only when a guest device failed to reach the internet. Repaired by re-pointing each orphan's Interface and NAT Address at the current tunnels, then deleting the duplicates, restoring the documented 13-rule set. Verification must be done with `pfctl -sn` rather than the UI, since the UI renders orphans as if they were live rules. Caution and procedure added to [`network/nat-rules.md`](network/nat-rules.md).
- Wi-Fi throughput on the trusted VLAN restored from 27 Mbps to 519 Mbps (2026-07-31). Wireless clients on the trusted VLAN had been capped near 30-95 Mbps while wired clients on the same VLAN ran at 890 Mbps. Every RF metric read healthy throughout: -52 dBm signal, 456 Mbps negotiated PHY rate, 3% channel utilization, 3% TX retries. Root cause was an 802.1Q tagging mismatch on the AP uplink: the controller had the SSID pointed at a VLAN-tagged network object while the switch port carries that VLAN untagged as its PVID. The AP transmitted tagged and the switch returned untagged, so return frames no longer matched the tagged WLAN and the AP fell back to software MAC bridging off its hardware-accelerated path. Traffic passed correctly the whole time, which is why nothing ever alarmed. The switch port map had always specified this SSID as untagged; the controller side had been built tagged, and the two were never reconciled. Fixed on the controller side by setting the SSID's network to `Native`; no switch change, because that VLAN also carries the AP's own untagged management address and tagging it would cut controller access. The other two SSIDs were tagged on both sides, correct, and unaffected. Diagnostic pattern worth reusing: testing a client on a different SSID on another VLAN, same AP and same moment, isolated the fault in two minutes after several hours spent on RF, cabling, PoE and client-side theories. Guidance added to [`network/switch-port-map.md`](network/switch-port-map.md).
- Suricata IDS false positives resolved at root cause after three incidents in six weeks (2026-07-31). The June and July fixes (disabling Block Offenders on two interfaces, then pass-listing one IP range) were both workarounds that left the source untouched. The diagnostic that settled it was measuring the alert distribution rather than chasing individual signatures: `grep -ohE '\[1:[0-9]+:' /var/log/suricata/*/alerts.log | sort | uniq -c | sort -rn`. Of roughly 980,000 recorded alerts, ~99.5% came from a single ruleset, `stream-events.rules`, spread across 22 distinct signature IDs with the top one at 553,515 hits. That ruleset flags TCP conditions that are abnormal on a clean LAN and entirely normal on a VPN-tunneled one (packet out of window, invalid ack, excessive retransmissions, FIN out of window, invalid timestamp). Since all traffic here crosses a WireGuard tunnel at MTU 1420 to an out-of-region exit, retransmission and reordering are the permanent normal condition of the path, and the rules fired hardest against the busiest destinations. That is why the block table filled with content delivery networks (object storage, streaming, search, mobile platforms) rather than attackers. Fix: `stream-events.rules` unchecked under Categories on all six Suricata interfaces, per-interface restart, block table flushed. Full policy in `docs/LIMITATIONS.md` and `operations/testing-procedures.md`.
- Threat detection rulesets enabled for the first time (2026-07-31). Audit during the above found that every `emerging-*.rules` category was unchecked on every interface, along with the Feodo Tracker and ABUSE.ch SSL Blacklist feeds. The only rules ever loaded were Suricata's built-in protocol anomaly set. The IDS had therefore been blocking legitimate CDN traffic on the basis of TCP stream shape while detecting zero actual threats. Nine categories enabled on WAN only as a staging step, since Block Offenders is disabled there and a false positive costs a log line rather than an outage: Feodo Tracker Botnet C2, ABUSE.ch SSL Blacklist, `emerging-botcc`, `emerging-compromised`, `emerging-exploit`, `emerging-malware`, `emerging-mobile_malware`, `emerging-phishing`, `emerging-worm`. Selection criterion locked: enable rules that match known-bad indicators, never rules that infer intent from traffic shape. Excluded on purpose: scan/dos/icmp (volume without value on a public IP), policy/info/games/p2p/chat/tor/user_agents (policy rather than security), web_*/sql (payload inspection, blind against tunneled HTTPS), retired/deleted/current_events (obsolete or high churn). Promotion to the blocking VLAN interfaces is gated on one week of clean WAN logs.
- WireGuard tunnel build checklist established after three latent faults were found on replacement interfaces (2026-07-31). All three were introduced when the previous two exits were deleted and rebuilt the day before; the new interfaces did not inherit the settings the old ones had, and none of the three produced a visible symptom that pointed at itself. Fault 1: the Tier 1 gateway's Monitor IP had been set to the firewall's own WireGuard interface address, so dpinger was pinging the local kernel, the gateway could never be marked Offline, and the failover group could never promote the other tier while the dashboard showed a healthy green Online throughout. Fault 2: the replacement tunnel interface was left at the default MTU 1500 instead of 1420, producing a PMTU black hole and throughput swinging between 240 and 900 Mbps on the same exit. Fault 3: one of the two configured DNS forwarders had no route and had never resolved, leaving the lab on a single live forwarder while the config claimed two, which is the same single point of failure that caused the 2026-06-04 LAN-wide DNS outage. Checklist now documented in `vpn/vpn-failover.md`.
- VPN tier order corrected by measurement (2026-07-31). Tier 1 was carrying all traffic at 68 ms while Tier 2 sat at 37 ms. Swapping the tiers in `VPN_FAILOVER` and resetting states held the average throughput at roughly 470 Mbps but raised the floor from 240 to 340 Mbps and halved the spread from 660 to 320 Mbps across 21 runs. Tier selection is now a measured decision (gateway RTT first, then a dozen throughput runs) rather than a choice made on regional reputation.
- DNS resilience 4-layer defense corrected after the 2026-07-31 findings. The Layer 1 forwarder table and the Layer 3 gateway monitor design both specified Mullvad in-tunnel `100.64.0.x` addresses. Testing proved those addresses receive no host route from pfSense and therefore never resolve. See the revised sections in `vpn/dns-resilience.md`.
- DNS resilience hardened end-to-end with a 4-layer defense (2026-06-04, same day as the outage). Layer 1: forwarder redundancy expanded from 2 single-vendor entries to 4 Mullvad entries (`100.64.0.31` / `100.64.0.32` family content + `100.64.0.1` base + `100.64.0.2` adblock), distributed across both VPN tier gateways so any single tunnel failure leaves the other half of the forwarder set alive. Layer 2: Unbound advanced settings tuned (Serve Expired enabled, Prefetch enabled, TTL for Host Cache Entries dropped from 15min to 5min, Aggressive NSEC enabled, Harden DNSSEC Data unchecked for consistency with the General-tab DNSSEC setting). Layer 3: Gateway monitor IPs moved from public Cloudflare/Quad9 (`1.1.1.1` / `9.9.9.9`) to Mullvad in-tunnel IPs (`100.64.0.31` for Tier 1, `100.64.0.32` for Tier 2) so the gateway monitor catches tunnel-internal failures, not just handshake liveness. Layer 4: cron-driven shell script (`/root/unbound_health.sh`) probes Unbound at the resolver loopback every 60s; 3 consecutive failures triggers an automatic `pfSsh.php playback svc restart unbound` that rebuilds wedged outbound sockets. Monit was the planned tool but is deprecated from the pfSense 2.8.x repo, so the cron+script approach was used instead. Maximum detection-to-recovery window is ~3 minutes; the 2026-06-04 outage would have self-recovered without human intervention under this stack. Full procedure, all code, all reasoning, and known limitations documented in [`vpn/dns-resilience.md`](vpn/dns-resilience.md). `vpn/vpn-failover.md` updated to reflect the new monitor IPs and to keep the previous public-IP design recorded for historical context.
- Resolved LAN-wide DNS outage and restored functionality (2026-06-04). Primary cause: pfSense Unbound entered a soft-hang state. Service status reported running but its UDP/53 sockets to the upstream forwarders had wedged after a recent WireGuard re-handshake. Plain service restart did not clear it; the outbound sockets persisted. Fix was a config-reload: added a redundant Mullvad DNS forwarder entry, which forced Unbound to rebuild its outbound sockets cleanly. After the rebuild, all configured forwarders started responding within ~60ms. Underlying design flaw documented for follow-up: prior setup had only two forwarders, both behind the same vendor, no redundancy and no health-check-driven auto-restart, single point of failure on DNS for the whole LAN. Hardening list tracked in Pending. Separately, a workstation showed periodic short ping drops (`General failure` bursts of 5-15s every few minutes). Initial hypothesis was a USB-Ethernet docking-station NIC flap; an A/B test with a spare USB-C adapter produced a noticeable but not conclusive reduction. The test was confounded because multiple variables had been changed in parallel. Investigation of the residual short drops continues and will be tracked locally; current top suspect is the previously-documented Win11 NIC bridge-filter pattern (see 2026-06-03 row in README).
- VPN exits migrated to dual USA (2026-06-01). Tier 1 active, Tier 2 failover. Selection: Tier 1 chosen by lowest WAN-to-endpoint latency (Diagnostics > Ping); Tier 2 chosen for geographic and peering diversity from Tier 1. Naming convention split: `TUN_` for the WireGuard Tunnel object, `PEER_` for the Peer sub-object (previously both were tagged `PEER_`). Sub-field descriptions standardized: Interface Address = `INT_USA_<N>`, Allowed IPs = `fulltunnel`. Public docs use tier numbers (`USA_1`, `USA_2`) only; exit cities are kept private per the global privacy rule. Old EU tunnels deleted in strict dependency order (DNS > Outbound NAT > Gateways > Interface assignments > Peers > Tunnels). Monitor IPs reset to canonical `1.1.1.1` (Tier 1) and `9.9.9.9` (Tier 2) post-cutover. Failover verified: disabling Tier 1 tunnel flips traffic to Tier 2 within ~30s, re-enable flips back. Zero leaks confirmed via mullvad.net/check and ipleak.net.
- U7 Lite AP pinned to static `10.10.10.254` via pfSense DHCP reservation on VLAN10_USERS (2026-05-29). Hostname `u7-lite-ap`. Mac Mini already at static `10.10.10.250`. Both within the `.201-.254` reserved range.
- OOB management port enabled on Protectli Port 3 / igb2 (2026-05-27). Subnet `172.16.99.0/24`, gateway `172.16.99.1`, DHCP `.10-.99`. Plug a laptop into Port 3 to get direct WebUI access bypassing the switch and the trunk entirely. New priority-one recovery path.
- VLAN 999 + dedicated management VLAN hardening attempted and abandoned (2026-05-27). GS308E v4 lacks a configurable Management VLAN feature per the official NETGEAR manual. Full post-mortem in `docs/LIMITATIONS.md`. Lab stays at 9/10 enterprise grade with the 1-point gap documented as hardware-bounded.
- GS308E Part 1 port allocation LOCKED (2026-05-27). Documented in `network/switch-port-map.md` with full Native VLAN + Tagged VLAN IDs format. Layout:
  - Port 1 UPLINK-PFSENSE (trunk, all VLANs)
  - Port 2 AP-TRUNK (U7 Lite, native 10, tagged 30 + 50)
  - Port 3 GS305-CHAIN (GS305 unmanaged downstream, VLAN 10)
  - Port 4 MAC-MINI (dedicated VLAN 10)
  - Port 5 PERSONAL-PC (dedicated VLAN 10)
  - Port 6 PRINTER (VLAN 20)
  - Port 7 SW1-CATALYST (Cisco lab uplink, VLAN 40; R1 plugs into SW1 directly)
  - Port 8 WAN-ESCAPE (VLAN 50)
- Cascaded the locked layout into `docs/technical-guide.md` §9.1/§9.2, `labs/ccna/README.md`, `labs/ccna/ccna-lab.md`, `services/mac-mini/README.md` (Mac VLAN 20 migration plan shelved).
- AP VLAN-tagged Wi-Fi root cause found (2026-05-27): the U7 Lite was tagging frames correctly all along. The downstream Netgear GS305 (unmanaged) in the path was stripping 802.1Q tags. Confirmed via tcpdump on the AP eth0 showing `vlan 30` tagged frames egressing the wire. Resolved by moving the AP cable to a direct GS308E port. Full session log in `keep_local/ap.md`.

### Pending (Part 1 close-out)
- Move native VLAN on Port 1 from `1` -> `999` (black-hole) for trunk hardening
- Label all 8 ports in the GS308E UI matching `switch-port-map.md`
- Export GS308E config and commit to repo for backup
- Physically execute the locked layout (move cables to match the port table)

### Done (earlier in Phase 3)
- Tailscale remote access, advertised routes for `10.10.1.0/24` and `10.10.10.0/24` configured and tested (March 2026)
- Suricata IDS deployed on pfSense 2.8.1
- pfBlockerNG-devel deployed for IP/DNS blocking
- pfBlockerNG DNSBL VIP activated on 10.10.99.1/32 Localhost (April 2026). Network wide ad and tracker DNS blocking now active on VLAN10_USERS, VLAN20_IOT, VLAN30_GUEST, and VLAN50_MGMT. Blocked lookups resolve to the VIP and serve the pfBlockerNG block page.
- pfBlockerNG DNSBL groups expanded (April 2026): ADs_Basic, ADs, Firebog_Suspicious, Phishing, BBcan177, Malicious, all set to Unbound action and daily updates. Verified `nslookup doubleclick.net` returns the DNSBL VIP.
- VLAN 10 firewall rule #1 narrowed (April 2026) from blanket `Pass any -> 10.10.10.1` to three scoped rules: UDP 53 (DNS), TCP 443 (WebUI), ICMP (gateway reachability).
- VLAN 50 firewall rule #1 narrowed (April 2026) from blanket `Pass any -> 10.10.50.1` to four scoped rules: UDP 53, TCP 443, TCP 22 (SSH), ICMP. Rule #7 `Pass any -> any` retained for the escape-hatch workflow.
- Tailscale ACLs locked down (April 2026): subnet approval reduced to `10.10.10.0/24` only, default-deny ACL with `tag:home` + `autogroup:admin` -> `10.10.10.0/24:*`, primary device tagged `tag:home`. VLAN 50 stays physical-only via switch port 8.
- Suricata enabled on VLAN 40 (April 2026) in alert-only mode. Suppression list deferred until lab gear comes online and Cisco-protocol SIDs (CDP, STP, DTP) can be observed.
- Backup procedure documented (April 2026), pfSense XML export and GS308E config backup with age encryption (`operations/backup-procedure.md`).
- Repeatable testing procedures documented (April 2026), VLAN isolation matrix, DNSBL verify, kill-switch drill, failover drill, leak tests (`operations/testing-procedures.md`).

### Fixed
- GS308E static DHCP moved from VLAN10 (wrong tab, wrong subnet) to LAN at `10.10.1.100`
- LAN DHCP range shrunk to `10.10.1.10-10.10.1.99` to avoid static mapping conflict
- VLAN 50 firewall docs updated from 2 rules to actual 5 rules
- VLAN 10 DHCP range corrected from .253 to .245 to match pfSense
- Clarified VLAN 50 WAN NAT behavior in docs
- Added temp switch management rule note to VLAN 10 firewall docs
- VLAN 10 rule #1 updated to pass any (pfSense UI access for lab work)
- VLANs 20, 30, 40 rule #1 locked down to UDP 53 only (DNS)
- DoH/DoT port range corrected to `443-853` across all VLANs
- Added missing LAN (VLAN 1) firewall rules section
- Added hardware images to repo structure diagram
- Renamed "Reserved For" column to "Purpose" in physical ports table

### Exploring
- Centralized logging, syslog server for firewall and IDS alerts

### On Hold
- AP hardware upgrade, VLAN-aware AP (TP-Link EAP or UniFi) not planned right now, may revisit
- Guest rate limiting on VLAN 30, deferred due to R6400 hardware limitations

---

## [Phase 2], 2026-03-17

### Added
- Router-on-a-Stick topology via single 802.1Q trunk (`igb1`)
- 6 VLANs: Native (1), Users (10), IoT (20), Guest (30), Lab (40), Management (50)
- Dual Mullvad WireGuard tunnels in a primary/failover gateway group
- `VPN_FAILOVER` gateway group with automatic failover
- Layered kill switch, manual outbound NAT with zero explicit WAN rules, DoH/DoT block, port 53 block, RFC1918 inter-VLAN isolation, IPv6 block
- DNS locked to Mullvad resolvers through VPN tunnels, no ISP fallback
- Netgear GS308E v4 configured with 802.1Q port assignments
- Netgear R6400 set to AP-only mode on VLAN 10
- Cisco lab gear isolated on VLAN 40
- Leak testing passed, ipleak.net and Mullvad connection check

### Changed
- Switch management moved to VLAN 50 only, removed VLAN 10 access

### Documented
- Full architecture writeup, firewall rules manual, and switch/VLAN manual (PDFs in `/docs`)
- Shareable config breakdowns in `/configs`

---

## [Phase 0], 2024 Origins

### Added
- Early 2024, Acquired Protectli FW6E, initial pfSense install with basic WAN/LAN configuration
- Later 2024, First Mullvad WireGuard tunnel, started learning VPN enforcement and outbound NAT discipline
- Through 2025, Added DNS and DoH blocking, built up early kill switch rules incrementally, refined VLAN segmentation
- Late 2025, Work transitioned into the polished Phase 2 build (dual tunnel failover, 6 VLAN topology, Tailscale, Suricata, pfBlockerNG-devel)

> **Note:** Phase 0 absorbs the original Phase 1 scope. Earlier drafts tagged this same work as "Phase 1, Early 2026" with incorrect dates. The real timeline is 2024 onward.

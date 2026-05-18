# CCNA Lab

A hands-on Cisco lab I'm building inside the home network to work through CCNA 200-301 topics on real hardware. Routing, switching, VLANs, ACLs, NAT, OSPF, all of it, configured from the console rather than just read about.

Isolated on VLAN 40 of the home lab so nothing the Cisco gear does can leak into the trusted network. Documented per phase so I can resume after breaks and so the show-run dumps survive a factory reset.

---

## Topology

![CCNA lab topology](topology/ccna.webp)

---

## Hardware

| Device | Role |
|---|---|
| Cisco Catalyst 3560 (`SW1`) | Layer 2/3 switch, access ports + inter-VLAN routing |
| Cisco 1900 ISR (`R1`) | Router, static routing, NAT, OSPF |
| Lab laptop | Sole CCNA client, plugged into `SW1` Fa0/3 |
| USB-to-RJ45 console cable (FTDI) | Console access, moved between `SW1` and `R1` as needed |

---

## Network Placement

Both Cisco devices uplink into the home lab's GS308E switch on access ports tagged for VLAN 40. The lab VLAN is firewall-isolated from the rest of the home network and only allowed egress is via the VPN failover gateway.

| Device | Home VLAN | Mgmt IP | Notes |
|---|---|---|---|
| pfSense (gateway) | 40 | 10.10.40.1/24 | Default route for the lab |
| `SW1` (3560) | 40 | 10.10.40.2/24 | Uplinked via GS308E port 6 |
| `R1` (1900) | 40 | 10.10.40.3/24 | Uplinked via GS308E port 7 |

Inside the Cisco gear, lab-internal VLANs (110/120/130) carry the actual CCNA exercises. Those subnets never leave the Cisco devices.

---

## Topics Covered

Phase-driven, so I work one CCNA concept at a time and dump the resulting `show run` after each.

| Phase | Topic | Status |
|---|---|---|
| 0-2 | Console, factory reset, baseline config | Done |
| 3 | Management IP, default gateway, ping test | In progress |
| 4-5 | VLANs 110/120/130 on `SW1`, DHCP pools on `R1` | Pending |
| 6-7 | Router-on-a-stick trunk, inter-VLAN routing | Pending |
| 8 | Static and default routes | Pending |
| 9 | OSPF single area | Pending (simulated in GNS3 if no second router) |
| 10-11 | ACLs, NAT/PAT | Pending |
| 12-13 | Port security, STP observation | Pending |
| 14 | EtherChannel | Simulated in Packet Tracer (need 2 switches) |
| 15-16 | SSH replacing Telnet, syslog to pfSense | Pending |

Full plan with target configs lives in [`ccna-lab.md`](ccna-lab.md).

---

## Folder Structure

```
ccna-lab/
├── README.md           # This file
├── ccna-lab.md         # Lab plan, IP scheme, baseline configs, phase progression
├── cable-labels.docx   # Printable cable label sheet
├── topology/           # Topology diagrams
├── show-runs/          # Per-phase `show running-config` dumps
└── screenshots/        # PuTTY captures, ping output, verification screenshots
```

---

## Documentation

- [`ccna-lab.md`](ccna-lab.md), full lab plan, IP plan, baseline config template, phase progression, house rules
- [`cable-labels.docx`](cable-labels.docx), printable cable label sheet to wrap around each cable end
- [`show-runs/`](show-runs/), per-device per-phase running config dumps (secrets stripped before commit)

---

## What I've Tested

- **VLAN 40 isolation holds**, lab gear cannot reach VLAN 10/20/30 hosts, confirmed by ping
- **Console connectivity**, both devices accessible at 9600 8-N-1 via FTDI cable
- **Factory reset and baseline config**, repeatable from this repo, password manager has the secrets

---

## Roadmap

| Item | Status | Notes |
|---|---|---|
| Phase 3 verification (ping matrix) | In progress | From `SW1`, `R1`, and lab laptop |
| Phase 4 VLAN buildout | Pending | Lab VLANs 110/120/130 on `SW1` |
| Second switch / router for multi-device topics | On hold | Currently planning to simulate Phases 9 + 14 in GNS3 / Packet Tracer |
| SSH + syslog (Phases 15-16) | Pending | Syslog target = pfSense, ties into the home lab's central logging plan |

---

## Known Limitations

**Multi-device topics need simulation.**
With one switch and one router, OSPF convergence (Phase 9) and EtherChannel (Phase 14) cannot be done on physical hardware. Both are covered in Packet Tracer or GNS3 instead, with a note in each show-run dump that the topology was virtual.

**No SSH yet.**
Telnet is the default on these IOS versions out of the box. SSH is deferred to Phase 15 once the keys, domain name, and crypto settings are configured. Until then, console-only management.

**Lab gear is loud on the wire when powered up.**
CDP, STP, and DTP traffic from the Cisco devices is visible on VLAN 40. Suricata on pfSense is in alert-only mode for VLAN 40, with Cisco-protocol SIDs to be tuned into a suppression list during active lab work.

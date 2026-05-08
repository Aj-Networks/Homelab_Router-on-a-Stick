# CCNA Lab

Cisco gear configuration for hands-on CCNA practice, integrated with the home lab via VLAN 40.

---

## Devices

| Device | Role | Console port | Uplink |
|---|---|---|---|
| Cisco Catalyst 3560 (`SW1`) | Layer 2/3 switch | RJ-45 (light blue), 9600 8-N-1 | GS308E port 6 (access, VLAN 40) |
| Cisco 1900 ISR (`R1`) | Router | RJ-45 (light blue), 9600 8-N-1 | GS308E port 7 (access, VLAN 40) |

---

## Topology

```
[Internet via pfSense (VLAN 40 → VPN_FAILOVER)]
                |
         GS308E port 6 / 7  (access mode, VLAN 40)
              |        |
        [SW1 3560]  [R1 1900]
              |        |
              | (trunk or routed link, varies per phase)
              |
        [Lab Laptop on Fa0/3]
```

---

## Console connection

| Setting | Value |
|---|---|
| Cable | USB-to-RJ45 console (FTDI chipset) |
| Terminal | PuTTY |
| Type | Serial |
| Speed | 9600 |
| Data / Stop / Parity / Flow | 8 / 1 / None / None |
| COM port | Whatever Windows assigns (check Device Manager) |

One console cable, moved between SW1 and R1 as needed.

---

## IP addressing plan

### Management (within home lab VLAN 40)

| Device | Management IP | Mask | Default gateway |
|---|---|---|---|
| pfSense (mother) | 10.10.40.1 | /24 | n/a |
| SW1 (3560) | **10.10.40.2** | /24 | 10.10.40.1 |
| R1 (1900) | **10.10.40.3** | /24 | 10.10.40.1 |
| pfSense DHCP pool | 10.10.40.10-10.10.40.254 | /24 | 10.10.40.1 |

Static IPs sit below the DHCP pool to avoid conflicts.

### Lab-internal VLANs (live only inside the Cisco gear)

To prevent any mental confusion with home VLAN IDs (1, 10, 20, 30, 40, 50), CCNA-lab VLANs use the 110-series. Tags are stripped at the GS308E uplink (access-mode port 6/7), so lab VLANs never leak to the home network.

| Lab VLAN | Name | Suggested subnet | Notes |
|---|---|---|---|
| 110 | LAB_USERS | 192.168.110.0/24 | Lab laptop default home |
| 120 | LAB_SERVERS | 192.168.120.0/24 | Future use |
| 130 | LAB_GUEST | 192.168.130.0/24 | Future use |

These subnets are local to the Cisco lab, not advertised to pfSense, not routable to the Internet through the home lab. To give them Internet, configure NAT/PAT on R1 (CCNA exercise, Phase 11).

---

## Phase progression

| Phase | Topic | Status |
|---|---|---|
| 0 | Console connectivity | Done |
| 1 | Factory reset both devices | Done |
| 2 | Baseline config (hostname, passwords, banner, no ip domain-lookup) | Done |
| 3 | Management IP on VLAN 40, default gateway, ping test | In progress (test from VLAN 10 deferred) |
| 4 | Create lab VLANs 110/120/130 on SW1, assign access ports | Pending |
| 5 | DHCP pools on R1 for lab VLANs | Pending |
| 6 | Router-on-a-stick trunk between SW1 and R1 | Pending |
| 7 | Inter-VLAN routing via R1 sub-interfaces | Pending |
| 8 | Static / default routes | Pending |
| 9 | OSPF single area | Pending (may need GNS3 second router) |
| 10 | ACLs (numbered + named) | Pending |
| 11 | NAT / PAT on R1 | Pending |
| 12 | Port security on SW1 access ports | Pending |
| 13 | Spanning Tree observation | Pending |
| 14 | EtherChannel | Use Packet Tracer (need 2 switches) |
| 15 | SSH replacing Telnet | Pending |
| 16 | Logging to syslog (point at pfSense) | Pending |

---

## Baseline config template

Applied to both devices in Phase 2. Hostname is the only field that differs (`SW1` for the switch, `R1` for the router).

```
enable
configure terminal
hostname SW1
no ip domain-lookup
enable secret <stored-in-password-manager>
service password-encryption
line console 0
 logging synchronous
 password <stored-in-password-manager>
 login
 exec-timeout 0 0
exit
line vty 0 4
 password <stored-in-password-manager>
 login
 transport input ssh
exit
banner motd # Authorized Access Only - Lab Device #
end
copy running-config startup-config
```

### Password storage policy

- **Enable secret:** unique, stored only in the password manager, different per device.
- **Console + VTY:** shared lab password, stored in the password manager. Same on both devices for ease of use during lab work.
- **Never** committed to this repository.

---

## Phase 3: Management IP (target state)

### SW1 (3560)

```
enable
configure terminal
interface vlan 1
 ip address 10.10.40.2 255.255.255.0
 no shutdown
exit
ip default-gateway 10.10.40.1
end
copy run start
```

Verify: `ping 10.10.40.1` from SW1 should succeed.

### R1 (1900)

```
enable
configure terminal
interface gi0/1
 ip address 10.10.40.3 255.255.255.0
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.10.40.1
end
copy run start
```

Verify: `ping 10.10.40.1` from R1 should succeed.

### Isolation sanity checks (run from each device)

| Command | Expected | Why |
|---|---|---|
| `ping 10.10.40.1` | Success | VLAN 40 gateway reachable |
| `ping 10.10.10.1` | Fail | Inter-VLAN block holds (RFC 1918 alias) |
| `ping 8.8.8.8` | Success | Egress via VPN_FAILOVER |

If all three behave as expected, the Cisco gear is correctly isolated.

---

## Lab laptop

| Item | Value |
|---|---|
| Connected to | SW1 Fa0/3 (LAB_USERS) |
| Role | Sole CCNA lab client |
| Used for | Verifying VLAN access, DHCP, ping, routing, ACLs |

One dedicated laptop on one access port covers ~75% of CCNA hands-on topics. Multi-switch and multi-router topics (STP failover, EtherChannel, OSPF convergence) are practiced in **Packet Tracer** or **GNS3** to avoid hardware sprawl.

---

## House rules

| Rule | Reason |
|---|---|
| `copy run start` after every successful change | Otherwise config evaporates on reboot |
| Save a `show run` text dump to `configs/ccna-lab/show-runs/<device>-<phase>.txt` after each phase | Restore reference. Strip passwords before committing. |
| Cisco ports plug only into GS308E port 6 or 7 | Avoid bridging the lab into trusted VLANs |
| Never run a Cisco DHCP server on `10.10.40.0/24` | pfSense DHCP owns this range; conflict would break VLAN 40 |
| Lab VLANs 110/120/130 stay inside the Cisco gear | Tags get stripped at the GS308E access-mode uplink |
| Power off Cisco gear when not actively learning | Reduces VLAN 40 noise (CDP/STP), saves power |

---

## Software stack

| Tool | Purpose |
|---|---|
| PuTTY | Console + SSH |
| Tftpd64 | TFTP server for IOS upgrades and config backup |
| WinSCP | SCP for IOS image transfers |
| Cisco Packet Tracer | Multi-device topics (STP, EtherChannel, OSPF convergence) |
| GNS3 / EVE-NG Community | Virtual routers for OSPF multi-area, BGP |
| Wireshark | Packet capture on the lab laptop |

---

## Study resources

| Resource | Type |
|---|---|
| Jeremy's IT Lab, full CCNA 200-301 (YouTube, free) | Video |
| Wendell Odom, CCNA 200-301 Official Cert Guide Vol 1 + 2 | Book |
| Boson ExSim-Max for CCNA | Practice exams |
| Anki deck (Jeremy's IT Lab) | Spaced-repetition memorization |

---

## Backup

Cisco device configs are exported via the existing backup workflow, see [`backup-procedure.md`](backup-procedure.md). For each device:

1. From console: `show running-config`, copy/paste output to text file
2. Strip the passwords (or encrypt with `age` per the same policy as the GS308E binary)
3. Optionally upload via TFTP to a workstation: `copy running-config tftp://10.10.40.X/SW1.cfg`

Restore = factory reset (Phase 1), then paste config, then `copy run start`.

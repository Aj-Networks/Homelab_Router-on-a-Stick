# Known Limitations and Lessons Learned

This document captures hardware and configuration limitations encountered during the Homelab build, the work attempted, and the lessons learned. Each entry is written as a post-mortem so future work avoids repeating the same effort.

---

## 1. GS308E v4 cannot host a separate management VLAN

**Date documented:** 2026-05-27

### Goal

Two enterprise hardening tasks were planned together:
- Set the trunk's native VLAN to an unused "black-hole" VLAN ID (999) to neutralize VLAN-hopping attacks (CIS Benchmark recommendation)
- Migrate switch management from VLAN 1 (10.10.1.100) to a dedicated, isolated VLAN 60 (10.10.60.100) so management traffic is separated from data VLANs

This would close the documented 1-point gap from a 9/10 to 10/10 enterprise-grade score on the smart-managed switching tier.

### What was attempted

1. Created VLAN 1 sub-interface in pfSense (`igb1.1`) to begin migrating management to tagged VLAN 1
2. Reassigned the pfSense LAN interface from the untagged parent `igb1` to the tagged sub-interface `igb1.1`
3. Created VLAN 60 sub-interface in pfSense (`igb1.60`)
4. Assigned a new pfSense interface `LAN_MGMT` with static IP `10.10.60.1/24`
5. Added a firewall rule on VLAN10_USERS allowing traffic to the new `10.10.60.0/24` management subnet
6. Created VLAN 60 on the GS308E with port 1 as a tagged member
7. Next planned step was to change the switch management IP from `10.10.1.100` to `10.10.60.100`

### What went wrong

During the LAN reassignment in pfSense (step 2 above), all switch management access was lost. A second cascading failure occurred during the addition of the new sub-interface (step 3), at which point the entire local network became unreachable. pfSense WebUI was no longer accessible through the normal user VLAN routing path, and all VLAN client devices lost connectivity.

### Recovery procedure used

A direct ethernet cable was connected from a recovery PC to the Protectli FW6E LAN port (hardware port 2, `igb1`). This out-of-band recovery path bypasses the switch entirely and provides direct access to the pfSense WebUI at `10.10.1.1`. The pre-attempt XML configuration was uploaded through Diagnostics > Backup & Restore, returning the lab to its known-good state.

This recovery procedure has been documented in project memory as the priority-one fallback for any future pfSense outage.

### Root cause

After exhaustive testing of every visible configuration option in the GS308E v4 device UI (System > Management, System > Maintenance, VLAN > 802.1Q Basic, VLAN > 802.1Q Advanced including VLAN Configuration, VLAN Membership, and Port PVID), a deep review of the GS308E v4 official user manual (NETGEAR document 202-12712-01, April 2024) confirmed the following:

- The GS308E v4 does not implement a configurable Management VLAN feature
- VLAN 1 is hardcoded as the default VLAN and cannot be deleted from the switch
- The switch management CPU communicates implicitly on whichever VLAN the management IP subnet maps to via the connected port's PVID, with no mechanism to bind management to a non-default VLAN explicitly
- There is no documented procedure, advanced menu, or hidden configuration option to relocate the management interface to a different VLAN

The hardware does not support the configuration required to complete this migration.

### Lesson learned

Smart-managed switches in this product tier (NETGEAR "Plus" line, equivalent tiers from other consumer vendors) intentionally omit the Management VLAN feature to differentiate from fully-managed enterprise switches. This is a hardware and firmware design boundary, not a configuration gap that can be worked around.

To implement a black-hole native VLAN and a dedicated management VLAN, a fully-managed switch is required. Candidates: Cisco Catalyst, Aruba CX, TP-Link Omada (with explicit Management VLAN support), or equivalent. This is a future hardware decision, not a configuration task.

The attempt also reinforced the importance of preserving the out-of-band recovery path. The pfSense LAN interface must remain on the parent `igb1` (untagged) interface to keep direct-cable recovery available; this rule is now documented in project memory.

### Current state

The lab continues to operate on native VLAN 1 with switch management on VLAN 1 at `10.10.1.100`. This configuration is functional and secure within the threat model of a single-occupant home network where physical access to the trunk port already implies full network compromise.

### Reattempt criteria

Revisit this hardening only when:
- The GS308E v4 is replaced with a switch that supports explicit Management VLAN configuration, OR
- A future GS308E firmware release adds the Management VLAN feature (unlikely given the product positioning)

### Files referenced

- `configs/switch-port-map.md`
- `configs/vlan-assignments.md`
- `keep_local/ap.md`
- `keep_local/backups/pfsense-2026-05-27-prevlan999.xml` (recovery backup used)
- `all_manuals/GS308Ev4.pdf` (NETGEAR document 202-12712-01)

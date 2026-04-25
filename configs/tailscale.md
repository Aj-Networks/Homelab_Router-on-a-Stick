# Tailscale Configuration

Remote access layer for the lab. Subnet router on pfSense, restrictive ACLs scoped to user VLAN only.

---

## Topology

| Role | Machine | OS | Purpose |
|---|---|---|---|
| Subnet router | `mother` | FreeBSD 15.0 (pfSense 2.8.1) | Advertises and routes the approved subnet |
| Client | `win-dec-2025` | Windows 11 25H2 | Primary remote-access device |

---

## Advertised vs Approved Routes

pfSense advertises all VLANs to Tailscale, but only the user VLAN is approved at the admin console. Approval is the gating layer — unapproved routes are not reachable from any tailnet device.

| Subnet | Advertised by pfSense | Approved | Why |
|---|---|---|---|
| 10.10.1.0/24 | Yes | No | Native trunk, no devices |
| 10.10.10.0/24 | Yes | **Yes** | User VLAN — only subnet that needs remote reach |
| 10.10.20.0/24 | Yes | No | IoT, no remote use case |
| 10.10.30.0/24 | Yes | No | Guest, no remote use case |
| 10.10.40.0/24 | Yes | No | Lab, isolated by design |
| 10.10.50.0/24 | Yes | No | Management — physical-only via switch port 8 |

**Why VLAN 50 stays unapproved:** the management VLAN is the escape-hatch for raw-WAN access via switch port 8. Exposing it remotely dilutes that physical-presence security model. Emergency admin still works through `10.10.10.1` (pfSense LAN-side WebUI) reached via Tailscale on the user VLAN.

> **Cleanup todo:** trim the pfSense Tailscale daemon advertisement so it stops broadcasting unused VLANs. Approval is already locked, so this is hygiene, not security.

---

## ACL Policy

Configured at `https://login.tailscale.com/admin/acls/file`.

```json
{
	"tagOwners": {
		"tag:home": ["autogroup:admin"],
	},
	"acls": [
		{
			"action": "accept",
			"src":    ["tag:home", "autogroup:admin"],
			"dst":    ["10.10.10.0/24:*"],
		},
	],
	"ssh": [],
}
```

| Element | Value | Reason |
|---|---|---|
| `tagOwners` | `tag:home` owned by admin | Single-tag policy for a single-user lab |
| ACL src | `tag:home` + `autogroup:admin` | Tagged devices plus admin-user fallback for recovery |
| ACL dst | `10.10.10.0/24:*` | Only the approved subnet, all ports |
| Default | Deny | Anything not explicitly accepted is dropped |
| `ssh` | `[]` | Tailscale SSH disabled |

---

## Device Tags

| Machine | Tag | Notes |
|---|---|---|
| `win-dec-2025` | `tag:home` | Primary remote device — gets full VLAN 10 reach |
| `mother` | (none) | Subnet router; no client traffic originates from it |

---

## Verification

Tested 2026-04-25 from `win-dec-2025` over phone-hotspot (off-LAN) with Tailscale enabled.

| Target | Expected | Result |
|---|---|---|
| `10.10.10.1:443` (pfSense WebUI) | Success | Pass |
| `10.10.50.1:443` (mgmt) | Fail (route not approved) | Pass |
| `10.10.20.1:53` (IoT DNS) | Fail (route not approved) | Pass |

---

## Operational Notes

- **Key expiry on `win-dec-2025`:** disabled (single-user lab, primary device).
- **Recovery path:** `autogroup:admin` is in the ACL src list so an untagged device logged in as the admin user still reaches `10.10.10.0/24`. Removes the lock-out risk if the tag is dropped.
- **Adding a new device:** apply `tag:home` from the admin console after the device joins the tailnet. Without the tag the device is admin-user-only via `autogroup:admin` and still works for the same destination set.
- **Adding a tighter access tier:** introduce `tag:admin` with a stricter destination set rather than relaxing `tag:home`. Keep the default-deny posture.

---

## Key Design Decisions

- **One subnet, one tag, default-deny** — minimum blast radius for a single-user homelab. Avoids premature complexity.
- **VLAN 50 stays physical** — preserves the port-8 escape-hatch model; remote admin goes through `10.10.10.1` instead.
- **Admin-user fallback in src** — recovery safety net if the device tag is removed or the device is replaced before being re-tagged.
- **Tailscale SSH off** — the SSH attack surface is intentionally not extended to tailnet identity until there is a use case.

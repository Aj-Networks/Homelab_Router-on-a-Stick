# Mac Mini home server

Apple Silicon home server running 24/7 as the lab's Docker host, remote-manageable hub, and future home for self-hosted services (UniFi controller, password vault, NAS, AI stack, etc.).

## Hardware

| Item | Value |
|---|---|
| Model | Mac Mini (M4 chip, 2024 / 2025) |
| OS | macOS 15 (Sequoia) |
| Role | Always-on home server, Docker host, services hub |
| Power | Mains, configured to auto-recover after a power failure |

## Network placement

| Item | Current | Target |
|---|---|---|
| VLAN | 10 (Users) - temporary | 20 (Servers / IoT) once unmanaged 5-port switch (TP-Link TL-SG105) is in place behind GS308E port 5 |
| MAC address | `xx:xx:xx:xx:xx:xx` (in password manager / pfSense reservation, not in this repo) | n/a |
| IP | `10.10.10.250` (DHCP reservation in pfSense, VLAN10_USERS tab) | `10.10.20.10` once moved to VLAN 20 |
| Hostname | `mac-mini-home` | same |

Reservation lives in **pfSense > Services > DHCP Server > VLAN10_USERS > Static Mappings**. When the Mac moves to VLAN 20, delete this reservation and recreate under the VLAN20 tab with IP `10.10.20.10`.

## Current state

| Built | Status |
|---|---|
| macOS Energy settings configured for 24/7 (no sleep, wake-on-network, auto power restart) | Done. See [`setup.md`](setup.md) |
| Screen Sharing (VNC) enabled with VNC password | Done. See [`remote-access.md`](remote-access.md) |
| Remote Login (SSH) enabled for admin user | Done. See [`remote-access.md`](remote-access.md) |
| Verified remote access from Windows PC via RealVNC Viewer + PowerShell SSH | Done |
| DHCP reservation in pfSense pinning Mac to `10.10.10.250` | Done |
| UniFi Network Application installed natively, adopted the U7 Lite AP (May 2026) | Done |

## Coming next

Planned phases. Each becomes its own subdoc when built, not before.

| Phase | File (not yet created) | Status | Topic |
|---|---|---|---|
| Docker Desktop install | `docker.md` | Pending | Container runtime for everything below |
| UniFi controller containerized | `services/unifi-controller.md` | Pending migration | Move from native macOS install to Docker container for restart/backup discipline |
| **AdGuard Home** | `services/adguard.md` | **Actively under review** | Network-wide DNS-based ad/tracker filtering, per-client profiles, query logs. Plan: container on Mac Mini, pfSense DHCP hands out Mac's IP as DNS to all clients |
| Vaultwarden | `services/vaultwarden.md` | Backlog | Self-hosted Bitwarden, password vault for lab + personal |
| AI stack | `services/ai-stack.md` | Backlog | Ollama / Open WebUI / local LLM containers |
| NAS | `services/nas.md` | Backlog | Time Machine target + SMB share for the household |
| Tailscale | extend `remote-access.md` | Backlog | Remote-from-anywhere overlay network |
| Backup | `backup.md` | Backlog | Time Machine policy, container volume backup, restore drills |

## Related docs

- [`../vlan-assignments.md`](../vlan-assignments.md) - VLAN map, where VLAN 20 fits
- [`../switch-port-map.md`](../switch-port-map.md) - GS308E port-by-port plan
- [`../firewall-rules.md`](../firewall-rules.md) - pfSense rules for inter-VLAN access to this host
- [`../r6400-setup.md`](../r6400-setup.md) - Current AP, marked for replacement by UAP-AC-PRO managed by this Mac
- [`../vpn-failover.md`](../vpn-failover.md) - VPN tunnels this host's traffic uses for outbound
- [`../tailscale.md`](../tailscale.md) - Tailscale overlay, future remote-access path
- [`../ccna-lab/`](../ccna-lab/) - CCNA lab, separate network segment (VLAN 40) but linked from this hub for cross-reference

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

| Item | Target | Status |
|---|---|---|
| VLAN | 20 (Servers / IoT) | Target, awaiting unmanaged 5-port switch (TP-Link TL-SG105 or similar) to share GS308E port 5 |
| Physical port | Behind a new dumb switch hanging off GS308E port 5 | Pending switch purchase |
| IP | `10.10.20.10` (DHCP reservation in pfSense) | Pending |
| Hostname | `mac-mini-home` (suggested) | Set in macOS sharing settings |

Until the dumb switch arrives, the Mac is on a temporary port for initial setup. Move to VLAN 20 once gear is in hand.

## Current state

| Built | Status |
|---|---|
| macOS Energy settings configured for 24/7 (no sleep, wake-on-network, auto power restart) | Done. See [`setup.md`](setup.md) |
| Screen Sharing (VNC) enabled with VNC password | Done. See [`remote-access.md`](remote-access.md) |
| Remote Login (SSH) enabled for admin user | Done. See [`remote-access.md`](remote-access.md) |
| Verified remote access from Windows PC via RealVNC Viewer + PowerShell SSH | Done |

## Coming next

Planned phases. Each becomes its own subdoc when built, not before.

| Phase | File (not yet created) | Topic |
|---|---|---|
| Docker | `docker.md` | Install Docker Desktop, configure autostart, allocate resources |
| UniFi controller | `services/unifi-controller.md` | Container for managing the UAP-AC-PRO (replaces R6400) |
| Vaultwarden | `services/vaultwarden.md` | Self-hosted Bitwarden, password vault for lab + personal |
| AI stack | `services/ai-stack.md` | Ollama / Open WebUI / local LLM containers |
| NAS | `services/nas.md` | Time Machine target + SMB share for the household |
| Tailscale | extend `remote-access.md` | Remote-from-anywhere overlay network |
| Backup | `backup.md` | Time Machine policy, container volume backup, restore drills |

## Related docs

- [`../vlan-assignments.md`](../vlan-assignments.md) - VLAN map, where VLAN 20 fits
- [`../switch-port-map.md`](../switch-port-map.md) - GS308E port-by-port plan
- [`../firewall-rules.md`](../firewall-rules.md) - pfSense rules for inter-VLAN access to this host
- [`../r6400-setup.md`](../r6400-setup.md) - Current AP, marked for replacement by UAP-AC-PRO managed by this Mac
- [`../vpn-failover.md`](../vpn-failover.md) - VPN tunnels this host's traffic uses for outbound
- [`../tailscale.md`](../tailscale.md) - Tailscale overlay, future remote-access path
- [`../ccna-lab/`](../ccna-lab/) - CCNA lab, separate network segment (VLAN 40) but linked from this hub for cross-reference

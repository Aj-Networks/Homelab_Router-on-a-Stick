# Mac Mini setup - macOS baseline

First-time macOS configuration for 24/7 server role. Done once after unboxing or factory reset.

## Energy settings (24/7 always-on)

**System Settings > Energy** (Apple Silicon Macs show this directly, no "Battery" tab since Mini has no battery)

| Toggle | Value | Reason |
|---|---|---|
| Low Power Mode | **OFF** | Full performance, no throttling under container load |
| Prevent automatic sleeping when display is off | **ON** | Critical. Without this the Mac sleeps, Docker pauses, services unreachable |
| Wake for network access | **ON** | Allows Wake-on-LAN and remote-access protocols to wake the host if it ever idles |
| Start up automatically after a power failure | **ON** | Mac auto-boots after outage, no manual intervention needed |

## Lock Screen settings (display only, system stays awake)

**System Settings > Lock Screen**

| Setting | Value | Reason |
|---|---|---|
| Start Screen Saver when inactive | Never (or 1 hour) | Display can sleep, that is fine for a headless server |
| Turn display off when inactive | 10 minutes | Saves the panel if you have a monitor attached; ignored if truly headless |
| Require password after screen saver begins | On, "after 5 seconds" | Physical security at the console |

## Sharing settings

**System Settings > General > Sharing**

| Toggle | Value | Reason |
|---|---|---|
| **Screen Sharing** | **ON** | VNC desktop access from other machines. See [`remote-access.md`](remote-access.md) |
| File Sharing | OFF (for now) | Will turn ON when NAS role is added |
| Media Sharing | OFF | Not needed |
| Printer Sharing | OFF | Printer is its own device on VLAN 20 |
| **Remote Login** | **ON** | SSH access. See [`remote-access.md`](remote-access.md) |
| Remote Management | **OFF** | Apple Remote Desktop, paid feature, conflicts with Screen Sharing if enabled |
| Remote Apple Events | OFF | Legacy AppleScript over network, security risk |
| Internet Sharing | OFF | Mac is not a router, pfSense owns that |
| Content Caching | OFF | Optional later for iOS update caching |
| AirDrop & Handoff | personal preference | Off for pure server, on if you use it from your phone/laptop |
| Advanced > Remote Application Scripting | OFF | Same reason as Remote Apple Events |

### Per-toggle detail (click the (i) next to each)

**Screen Sharing > (i):**
- Allow access for: **Only these users** → add your admin user account explicitly (do not rely on the "Administrators" group)
- Anyone may request permission to connect: OFF
- **VNC viewers may control screen with password: ON** → set a strong password, save it in your password manager as "Mac Mini VNC password" (separate from Mac login password)

**Remote Login > (i):**
- Allow access for: **Only these users** → add your admin user account
- Allow full disk access for remote users: OFF (security best practice)

## Security baseline

| Item | Setting | Notes |
|---|---|---|
| FileVault disk encryption | ON | System Settings > Privacy & Security > FileVault. Required for any machine that might leave the property |
| Firewall | ON | System Settings > Network > Firewall. Allow incoming for Screen Sharing, Remote Login automatically when those toggles are on |
| Auto-update macOS | OFF | Set to download but NOT auto-install. Server updates should be scheduled manually to avoid surprise reboots breaking services |
| Login password | Strong, unique | Stored in password manager |

## Naming and identity

**System Settings > General > About:**
- Name: `mac-mini-home` (suggested) - this is the hostname seen on the network

**System Settings > General > Sharing > Local hostname (under Advanced):**
- Confirm it matches the device name. Should resolve as `mac-mini-home.local` via Bonjour from other devices on the same subnet.

## Verification

After setup is complete:
- [ ] Mac runs continuously for 24 hours without sleeping. Verify with `pmset -g log | grep -i sleep` from Terminal
- [ ] SSH login from another machine works (see [`remote-access.md`](remote-access.md))
- [ ] VNC desktop access from another machine works
- [ ] After an intentional power-yank test, Mac boots itself back up unattended within 60 seconds

## Reset / re-do procedure

If a clean slate is needed:
1. **System Settings > General > Transfer or Reset > Erase All Content and Settings**
2. Walk through Setup Assistant
3. Re-apply all settings from this doc top to bottom
4. Verify with the checklist above

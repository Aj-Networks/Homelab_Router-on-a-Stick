# Mac Mini remote access

How to reach the Mac Mini from another machine on the LAN. Both command-line (SSH) and full desktop (VNC) are covered. All free, no subscriptions.

## Prerequisites

Both must be ON in **System Settings > General > Sharing**:
- Remote Login (SSH)
- Screen Sharing (VNC), with "VNC viewers may control screen with password" enabled and a password set

See [`setup.md`](setup.md) for the full Sharing config.

## Connection details

| Protocol | Port | What it gives you |
|---|---|---|
| SSH | 22 | Command line (Terminal-style) |
| VNC | 5900 | Full graphical desktop |

Mac's current IP: **`10.10.10.250`** (pinned by DHCP reservation in pfSense, VLAN10_USERS tab). Target after VLAN 20 migration: `10.10.20.10`.

Verify with:
- On the Mac: `ipconfig getifaddr en0` in Terminal
- In pfSense: Status > DHCP Leases

## SSH from Windows

### Method A - built-in OpenSSH (recommended)

Windows 10/11 ships with `ssh`. From PowerShell or Windows Terminal:

```powershell
ssh <mac-username>@<mac-ip>
```

Example:
```powershell
ssh root@10.10.10.23
```

First connection prompts to accept the host fingerprint → type `yes`, then enter the macOS login password.

Find the exact Mac username by running `whoami` in Terminal on the Mac.

### Method B - PuTTY

If you prefer a GUI client:
- Host Name: the Mac IP
- Port: 22
- Connection type: SSH
- Save the session as `mac-mini-home` for one-click future use

## VNC from Windows (full desktop)

### Client: RealVNC Viewer (free)

- Download: https://www.realvnc.com/en/connect/download/viewer/
- Free for personal use. No account required.
- Alternatives if you want fully open source: TigerVNC Viewer, TightVNC, UltraVNC

### Connection steps

1. Open RealVNC Viewer
2. In the address bar at the top, type the Mac IP (e.g. `10.10.10.23`) and press Enter
3. RealVNC may warn about an unencrypted connection - this is normal for the macOS built-in VNC server. Continue.
4. Authentication prompt appears asking for credentials for the remote device:
   - **Username:** leave blank
   - **Password:** the VNC password you set in **Screen Sharing > (i) > VNC viewers may control screen with password**
   - Click OK
5. Mac desktop appears in a window. Control it like you would in person.

### Troubleshooting

| Symptom | Fix |
|---|---|
| Connection refused | Screen Sharing is off. Turn it on in System Settings > Sharing. |
| Password rejected | You typed your Mac login password instead of the separate VNC password. Use the one set under "VNC viewers may control screen with password". |
| Black screen | Mac is in display-only sleep mode. Move the mouse in the VNC window to wake the display. |
| Slow / laggy | Lower image quality in RealVNC: View menu > Quality > Medium. |

## Tailscale (future - remote from outside the LAN)

When set up, Tailscale lets you SSH and VNC into the Mac from anywhere (coffee shop, work, vacation) over an encrypted overlay network, no port forwards, no public IP needed.

To be added when configured. See [`../tailscale.md`](../tailscale.md) for the existing lab Tailscale setup that this will join.

Planned use:
- Install Tailscale macOS app on the Mac Mini, sign in, enable as a node
- Install Tailscale on each client device (Windows PC, phone, etc.)
- SSH/VNC to the Mac's Tailscale IP (`100.x.y.z`) from anywhere

## Credentials checklist

| Credential | Where stored |
|---|---|
| Mac login password (user account) | Password manager |
| VNC password (Screen Sharing) | Password manager, separate entry labeled "Mac Mini VNC password" |
| SSH key (optional, future) | If switched to key-based SSH, private key in `~/.ssh/` on each client + passphrase in password manager |

Never commit any of these to git.

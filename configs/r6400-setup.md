# Netgear R6400 - AP setup and recovery

The R6400 runs in **AP mode only** (routing/NAT/DHCP disabled, pfSense owns those). This doc is the restore procedure when a factory reset is needed.

## Target state

| Item | Value |
|---|---|
| Operating mode | Access Point (AP mode) |
| Management IP | `10.10.10.254` |
| Subnet mask | `255.255.255.0` |
| Gateway | `10.10.10.1` (pfSense VLAN 10) |
| Primary DNS | `10.10.10.1` |
| VLAN | 10 (Users / Trusted) |
| Switch port | GS308E port 3, access mode, PVID 10 |
| pfSense DHCP reservation | LAN MAC of R6400 -> `10.10.10.254` |
| WiFi security | WPA2-PSK [AES] on both 2.4 GHz and 5 GHz |
| Guest network | Enabled, "allow access to local network" = OFF (AP-level isolation) |

Cross-reference: [`vlan-assignments.md`](vlan-assignments.md), [`switch-port-map.md`](switch-port-map.md).

Secrets (admin password, WiFi password, SSIDs) live in the password manager, NOT in this repo.

## When to factory reset

Reset only if the GUI is unreachable AND power-cycling does not fix it. Common trigger seen in this lab: a stuck WiFi-off state where the GUI loads but the radio toggle does not respond. Symptoms:

- WiFi LEDs off
- "Enable Wireless Router Radio" checkbox does not stick after Apply
- Cannot reach AP from VLAN 10 even though port 3 link light is on

If those match, reset.

## Reset procedure

1. Hold the **Reset** button on the back with a paperclip for **7+ seconds** until the power LED blinks amber.
2. Wait 2 minutes for full reboot. AP comes back at `192.168.1.1` with default `admin` / `password`.

## Bench reconfigure (NOT plugged into the live network yet)

1. Plug a laptop into a **yellow LAN port** on the R6400 (not the blue WAN/Internet port).
2. Laptop NIC = DHCP. Should pull `192.168.1.x`.
3. Browse `http://192.168.1.1`. Login `admin` / `password`. Skip the setup wizard.

### Set new admin password
**Advanced > Administration > Set Password** - use value from password manager.

### Enable AP mode + fixed IP
**Advanced > Advanced Setup > Wireless AP**

- Check **Enable AP Mode**
- Select **Use fixed IP address**
- IP `10.10.10.254`, mask `255.255.255.0`, gateway `10.10.10.1`, DNS `10.10.10.1`
- Apply (AP reboots, ~90s)

After reboot, the laptop loses its `192.168.1.x` DHCP. Set laptop NIC to static `10.10.10.50/24`, browse `http://10.10.10.254`.

### WiFi - main SSID (both bands)

| Setting | Value |
|---|---|
| Enable Wireless Radio | ON |
| SSID | from password manager |
| Region | United States |
| Channel | Auto |
| Security | **WPA2-PSK [AES]** (avoid WPA3, R6400 firmware is flaky) |
| Password | from password manager |
| Broadcast SSID | ON |

### WiFi - guest network (both bands)

| Setting | Value |
|---|---|
| Enable Guest Network | ON |
| SSID | from password manager |
| Security | WPA2-PSK [AES] |
| Password | separate from main, from password manager |
| Allow guest to access my local network | **OFF** |

### Disable everything else (AP hygiene)

| Section | Toggle | Value |
|---|---|---|
| Setup > LAN Setup | Use Router as DHCP Server | OFF |
| Advanced Setup > UPnP | Turn UPnP On | OFF |
| Advanced Setup > IPv6 | Internet Connection Type | Disabled |
| Wireless > Advanced | Enable WPS | OFF |
| Administration > Remote Management | Turn Remote Management On | OFF |
| ReadyShare | All toggles | OFF |
| Parental Controls | Enable Live Parental Controls | OFF |
| Advanced Setup > Traffic Meter | Enable Traffic Meter | OFF |
| Advanced Setup > Dynamic DNS | Use a Dynamic DNS Service | OFF |

### Save a config backup
**Administration > Backup Settings > Back Up**. Save the `.cfg` to OneDrive or password manager. **Do NOT commit it to git** - it contains the WiFi password.

## Deploy

1. Power down R6400.
2. Move ethernet from bench laptop to **GS308E port 3** (use a yellow LAN port on R6400, NOT WAN).
3. Power up R6400, wait 90s.

## Pin the IP in pfSense (belt + suspenders)

Even with the R6400's fixed-IP setting, also reserve in pfSense so a future reset does not strand the AP at a random pool IP.

1. Get the R6400 **LAN MAC** from one of:
   - R6400 GUI: Advanced > Administration > Router Status > LAN Port MAC
   - pfSense: Status > DHCP Leases > VLAN10 (find the AP)
   - Bottom label of the device (use LAN MAC, not WAN MAC)
2. pfSense > Services > DHCP Server > **VLAN10** tab > Static Mappings > Add
   - MAC: the LAN MAC
   - IP: `10.10.10.254`
   - Hostname: `r6400-ap`
   - Description: `Netgear R6400 access point`
3. Save, Apply Changes.

## Verify

From a VLAN 10 device:

```
ping 10.10.10.254          # should reply
```

Then:
- `http://10.10.10.254` - AP GUI loads
- Connect a phone to the trusted SSID, internet works
- Connect a phone to the guest SSID, internet works, cannot ping `10.10.10.x` LAN devices

## Known issue log

| Date | Symptom | Fix |
|---|---|---|
| 2026-05 | WiFi toggle stuck off, GUI accessible but radio would not enable. AP also drifted to `10.10.10.32` after a partial reset. | Full 7-second factory reset, full bench reconfigure per this doc, pinned MAC in pfSense reservation. |

## Related

- VLAN 10 firewall context: [`firewall-rules.md`](firewall-rules.md)
- Why the R6400 is on VLAN 10 with AP-level guest isolation (and the limitation that creates): [`../docs/LAB_TECHNICAL_GUIDE.md`](../docs/LAB_TECHNICAL_GUIDE.md) section 25

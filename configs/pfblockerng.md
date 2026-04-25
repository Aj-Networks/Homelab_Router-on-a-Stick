# pfBlockerNG DNSBL Configuration

DNS sinkhole and ad and tracker blocking layer on pfSense 2.8.1.

---

## DNSBL Virtual IP

| Setting | Value |
|---|---|
| IPv4 VIP | 10.10.99.1 |
| IPv6 VIP | (blank) |
| Type | IP Alias |
| Interface | Localhost |
| Address | 10.10.99.1/32 |
| Description | pfBlockerNG DNSBL VIP |

### Why 10.10.99.0/24

Chosen because it does not overlap any active VLAN (1, 10, 20, 30, 40, 50) or the Tailscale advertised routes (10.10.1.0/24 and 10.10.10.0/24). 10.10.10.1 was rejected since it is the VLAN 10 Users gateway.

---

## DNSBL Settings

| Setting | Value |
|---|---|
| Web Server Interface | Localhost |
| Global Logging/Blocking Mode | DNSBL WebServer/VIP |
| Permit Firewall Rules | Enabled |
| Blocked Webpage | dnsbl_default.php |
| Resolver Cache | Enabled |

---

## Active Feeds

Groups are configured under `Firewall > pfBlockerNG > DNSBL > DNSBL Groups` and `Firewall > pfBlockerNG > IP > IPv4`.

### DNSBL Groups (DNS-level blocking)

| Group | Action | Frequency | Purpose |
|---|---|---|---|
| ADs_Basic | Unbound | Once a day | Core ad/tracker blocking |
| ADs | Unbound | Once a day | Extended ad/tracker coverage |
| Firebog_Suspicious | Unbound | Once a day | Suspicious domains (Firebog curated) |
| Phishing | Unbound | Once a day | Phishing domains |
| BBcan177 | Unbound | Once a day | Curated threat domains |
| Malicious | Unbound | Once a day | Malware domains (D_Me_Malv, D_Me_Malw, MVPS, SWC) |

### IPv4 Groups (IP-level blocking)

| Group | Feeds enabled | Action | Purpose |
|---|---|---|---|
| PRI1_Spamhaus | Spamhaus_DROPv4, Spamhaus_EDROPv4 | Deny Both | Hijacked / unallocated netblocks |
| PRI1_Botnet | Feodo_Tracker | Deny Both | Active botnet C2 IPs |
| PRI1_ET | ET_Compromised | Deny Both | Emerging Threats compromised hosts |

### GeoIP

Currently disabled. Outbound is already tunneled via Mullvad VPN; inbound is blocked by default. Can be enabled later if regional inbound blocking becomes a need.

---

## Update Schedule

Configured under `Firewall > pfBlockerNG > Update`.

| Setting | Value |
|---|---|
| CRON | Daily at 03:00 |
| Save | Enabled |
| Force flag | Off (let CRON run) |

After any feed/group change, force a manual reload from the `Update` tab: select `Reload`, mode `All`, click `Run`.

---

## Permit Firewall Rules

pfBlockerNG auto generates floating rules to let clients reach the VIP for the block page. Interfaces selected:

| Interface | Included | Reason |
|---|---|---|
| VLAN10_USERS | Yes | Client VLAN |
| VLAN20_IOT | Yes | Client VLAN |
| VLAN30_GUEST | Yes | Client VLAN |
| VLAN50_MGMT | Yes | Client VLAN |
| VLAN40_LAB | No | Isolated by design |
| VPN_CHI | No | Outbound tunnel |
| VPN_NYC | No | Outbound tunnel |
| LAN | No | Unused |

---

## Verification

Confirmed working April 2026.

- Bell icon notice cleared after Update, Force, Reload, DNSBL, Run.
- nslookup doubleclick.net from a VLAN 10 client returned 10.10.99.1.
- Browser rendered the pfBlockerNG block page.
- Alert logged: Client 10.10.10.10, Type DNSBL, Group ADs_Basic.

# Testing Procedures

Repeatable checks to verify the lab still behaves the way the docs say it does. Run these after any firewall, VLAN, NAT, or VPN change.

---

## 1. VLAN isolation matrix

From a test client on each VLAN, ping and nmap every other VLAN's gateway and a known host. Record actual vs expected.

| From VLAN | To 10.10.1.1 | To 10.10.10.1 | To 10.10.20.1 | To 10.10.30.1 | To 10.10.40.1 | To 10.10.50.1 |
|---|---|---|---|---|---|---|
| 10 | blocked | allowed (rule #1) | blocked | blocked | blocked | blocked |
| 20 | blocked | blocked | allowed (UDP 53 only) | blocked | blocked | blocked |
| 30 | blocked | blocked | blocked | allowed (UDP 53 only) | blocked | blocked |
| 40 | blocked | blocked | blocked | blocked | allowed (UDP 53 only) | blocked |
| 50 | allowed | allowed | allowed | allowed | allowed | allowed (rule #1) |

Commands per client:

```
ping -c 2 10.10.X.1
nmap -sT -p 22,53,80,443 10.10.X.1
```

**Expected on blocked paths:** no ICMP reply, all TCP shows `filtered` (pfSense drops, doesn't reset).

---

## 2. DNSBL verification

From any pfBlockerNG-enabled VLAN (10, 20, 30, 50) - see [pfblockerng.md](pfblockerng.md).

```
dig doubleclick.net @10.10.X.1
dig googleadservices.com @10.10.X.1
```

**Pass:** both return `10.10.99.1`.

Then in a browser, visit `http://doubleclick.net` - the pfBlockerNG block page should render.

From VLAN 40 (excluded):

```
dig doubleclick.net @10.10.40.1
```

**Pass:** returns the real IP (not sinked). Confirms VLAN 40 bypass is intact.

Check alerts:

```
Firewall > pfBlockerNG > Alerts
```

Should show a DNSBL hit matching your test client IP.

---

## 3. DoH / DoT block

From any client VLAN, try a DoH resolver directly:

```
curl -s -H 'accept: application/dns-message' 'https://1.1.1.1/dns-query?name=example.com&type=A' -o /tmp/resp
```

**Pass:** connection times out or is refused (port 443 to `DOH_IPS` alias is blocked - see [firewall-rules.md](firewall-rules.md) rule #3 on VLAN 10, rule #2 on VLANs 20/30/40).

---

## 4. Plain DNS leak test

From a client, try an external resolver:

```
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
```

**Pass:** times out. Only the local gateway answers DNS.

---

## 5. VPN kill switch drill

**Only do this on a test client, not your main machine.**

1. Note current IP at `ipleak.net` from the client - should be a Mullvad exit
2. On pfSense: `Interfaces > VPN_CHI` - uncheck `Enable`
3. Within 30s, gateway group should promote `VPN_NYC`
4. Refresh `ipleak.net` - should still be a Mullvad exit (NYC this time)
5. Now disable `VPN_NYC` too
6. Refresh - browser should fail to load anything; no IP leak
7. Re-enable both interfaces; normal service should resume inside a minute

**Pass:** at no point does the client's real WAN IP appear. Screenshot the `ipleak.net` result before/during/after for the record.

---

## 6. VPN failover drill (softer version of #5)

1. On pfSense: `Status > Gateways` - note `VPN_CHI` is `Online` and primary
2. Suspend the CHI WireGuard peer via `VPN > WireGuard` (toggle off)
3. Gateway group should mark CHI `Offline`, NYC `Online` and primary
4. From a client: `curl ifconfig.me` - should show a NYC Mullvad IP
5. Re-enable CHI; it should reclaim primary within ~30s

---

## 7. IPv6 block verification

From a client:

```
curl -6 -s https://ipv6.google.com
ping6 -c 2 ::1  # loopback only, should work locally but not leave
```

**Pass:** external IPv6 fails (block rule #6/7 per VLAN). Local IPv6 still works where applicable.

---

## 8. Tailscale reach

From a remote Tailscale device:

```
tailscale ping 10.10.10.X    # should succeed per advertised routes
ssh 10.10.50.1               # currently succeeds - flag for ACL tightening
```

Record what reaches what; use this to drive the Tailscale ACL policy (see [tailscale.md](tailscale.md) when it exists).

---

## 9. Suricata health

```
Services > Suricata > Interfaces - confirm LAN, VLAN10, 20, 30, 50 are UP (VLAN 40 intentionally skipped)
Services > Suricata > Alerts - alerts accumulating, not zero
```

**Pass:** alert counter moves during normal browsing (should see at least a few policy-info alerts in a day).

---

## 10. Leak tests - full external

Visit all of these from one client, capture screenshots:

- https://ipleak.net
- https://mullvad.net/en/check
- https://browserleaks.com/dns
- https://browserleaks.com/webrtc

**Pass:** all show Mullvad exit IPs, Mullvad DNS, no WebRTC leak of the real LAN IP.

---

## Cadence

| When | Tests |
|---|---|
| After any firewall or NAT rule change | 1, 3, 4 |
| After any VPN config change | 5, 6, 10 |
| After any pfBlockerNG feed/update | 2 |
| After any Tailscale change | 8 |
| Monthly (no changes) | 1, 2, 5, 10 |
| Before pfSense firmware upgrade | All |
| After pfSense firmware upgrade | All |

---

## Results log

Record pass/fail and the date. A test suite that isn't logged is a test suite that wasn't run.

| Date | Test # | Pass/Fail | Notes |
|---|---|---|---|
| _pending_ | _pending_ | _pending_ | _pending_ |

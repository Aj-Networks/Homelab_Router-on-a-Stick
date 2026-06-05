# DNS Resilience

Multi-layered defense against pfSense Unbound resolver failures when the upstream DNS forwarders ride a VPN tunnel that can transiently fail or wedge.

This document describes the verified 4-layer setup applied 2026-06-04 after a LAN-wide DNS outage caused by a resolver soft-hang.

---

## What it is

Four independent layers that each handle a different failure mode for the DNS path:

| Layer | Defends against | Mechanism |
|---|---|---|
| 1 | Single forwarder dead | Multiple Mullvad DNS IPs as upstream forwarders |
| 2 | Slow / stuck forwarder | Unbound advanced timing and caching tuning |
| 3 | Tunnel up but internal services dead | Gateway monitor IPs that live inside the tunnel |
| 4 | Resolver process up but sockets wedged | Cron-driven health check + Unbound auto-restart |

A single failure in any one layer is absorbed by the layers below it. The full stack survives the failure mode seen on 2026-06-04 (Unbound soft-hang after a WireGuard re-handshake left upstream UDP/53 sockets wedged), and it self-recovers within 3 minutes with no human intervention.

---

## Why

### The incident

Yesterday night, the LAN lost DNS. Symptoms:

- pfSense `Status → Services` showed Unbound **running** (process up, port bound)
- pfSense `Status → Gateways` showed both VPN tunnels **Online, 0% loss** to the public monitor IPs (`1.1.1.1`, `9.9.9.9`)
- pfSense `Diagnostics → DNS Lookup` for any hostname returned "Host could not be resolved", with all configured upstream forwarders showing **"No response"**
- Restarting the Unbound service from `Status → Services` did **not** clear it
- Apps relying on DNS (browser, OneDrive, peer-to-peer file sharing) failed

### Root cause

Unbound's outbound UDP/53 sockets to the Mullvad DNS forwarders had wedged. The wedge was triggered by a WireGuard tunnel re-handshake earlier that night. The re-handshake briefly disrupts the tunnel's underlying interface state; Unbound's existing open UDP sockets to upstream forwarders entered a stuck state from which a plain service restart did not recover.

This failure mode is sometimes called a **soft hang**: the process is alive, health checks pass, the service does not serve.

### Why service status alone is not enough

Three monitors were green during the incident:
- Service status (Unbound running)
- Gateway status (tunnels Online)
- Public DNS reachability (ICMP to 1.1.1.1 / 9.9.9.9 succeeded)

None of those touched the actual failed component (Unbound's stuck outbound sockets). Each was monitoring a layer that was healthy while the real failure sat below it.

### What actually recovered DNS

Adding a new DNS forwarder entry to `System → General Setup → DNS Servers` and saving. The act of saving forced Unbound to **reload its configuration**, which tore down and rebuilt the wedged outbound sockets. After reload, all forwarders responded normally within ~60ms.

The fix took less than a minute once identified, but identifying it took ~45 minutes because the symptom was upstream of the visible signals.

### Design conclusion

A resilient DNS stack must:
1. Have redundancy at the forwarder layer so a single dead forwarder is absorbed transparently
2. Probe and timeout aggressively so dead forwarders are detected fast
3. Monitor end-to-end success (resolution actually working), not just liveness signals (service up, ports bound, ICMP works)
4. Automatically remediate (reload / restart) when end-to-end monitoring fails

---

## Layer 1: Forwarder redundancy

`System → General Setup → DNS Servers`

Configured 4 Mullvad DNS forwarder entries spread across both VPN tier gateways:

| # | Mullvad IP | Hostname | Gateway | Mullvad tier |
|---|---|---|---|---|
| 1 | `100.64.0.31` | `family.dns.mullvad.net` | `GW_USA_1` | Family content (primary) |
| 2 | `100.64.0.32` | `family.dns.mullvad.net` | `GW_USA_2` | Family content (primary, other gateway) |
| 3 | `100.64.0.1` | `base.dns.mullvad.net` | `GW_USA_1` | No filter (fallback) |
| 4 | `100.64.0.2` | `adblock.dns.mullvad.net` | `GW_USA_2` | Ad block (fallback) |

`DNS Server Override`: unchecked.
`DNS Resolution Behavior`: Use local DNS (127.0.0.1), fall back to remote DNS Servers (Default).

### Why this distribution

- **Two filter tiers (family + fallbacks).** Unbound queries the fastest responder. The family-tier forwarders normally win because they are configured first and have a slight latency edge. If family-tier forwarders go dark together, the fallback tier still resolves. Filter consistency takes a small hit during fallback events; that trade-off is accepted in exchange for full resilience.

- **Each gateway used by 2 entries.** If `GW_USA_1`'s tunnel wedges, entries 2 and 4 (both on `GW_USA_2`) still resolve. Symmetric for `GW_USA_2`.

- **All entries stay inside Mullvad's CGNAT range (`100.64.0.0/10`).** No clearnet DNS, no leak to ISP. DNS privacy posture preserved.

### pfSense constraint

`System → General Setup → DNS Servers` rejects duplicate IPs even when gateways differ. Each row's IP must be unique. That blocks the "same IP via both gateways" pattern; the multi-tier approach above works around it by using different per-tier IPs.

---

## Layer 2: Unbound advanced tuning

`Services → DNS Resolver → Advanced Settings`

Changes from default:

| Setting | Default | New value | Effect |
|---|---|---|---|
| Serve Expired | Unchecked | **Checked** | Clients receive stale-but-valid cached answers if every forwarder is briefly unreachable, while Unbound refreshes in the background. Single biggest resilience win. |
| Prefetch Support | Unchecked | **Checked** | Popular cache entries refresh before expiry. Cache stays warm; slightly more upstream traffic. |
| TTL for Host Cache Entries | 15 minutes | **5 minutes** | Forwarders previously marked "down" are re-probed sooner after they come back. |
| Aggressive NSEC | Unchecked | **Checked** | Synthesizes NXDOMAIN responses from DNSSEC cache for known-nonexistent names. Reduces upstream traffic. |
| Keep Probing | (Checked by default) | unchanged | Unbound continues probing forwarders even after they are marked down. Critical for fast recovery once a forwarder recovers. |
| Harden DNSSEC Data | Was checked | **Unchecked** | Must match the DNSSEC support setting on the General Settings tab. Enabled in error previously, removed for consistency. |

All other Advanced Settings left at defaults.

---

## Layer 3: Gateway monitor on in-tunnel IPs

`System → Routing → Gateways`

Each Mullvad gateway's Monitor IP changed from a public-internet IP to a Mullvad-internal IP that only responds when the tunnel's internal services are healthy:

| Gateway | Old Monitor IP | New Monitor IP |
|---|---|---|
| `GW_USA_1` | `1.1.1.1` (Cloudflare) | `100.64.0.31` (Mullvad DNS, family tier) |
| `GW_USA_2` | `9.9.9.9` (Quad9) | `100.64.0.32` (Mullvad DNS, family tier) |

### Why

Public IPs (`1.1.1.1`, `9.9.9.9`) reach their destination over the WireGuard tunnel and out Mullvad's WAN. They report Online as long as the tunnel handshake is alive AND Mullvad's general egress works. They do NOT detect the case where Mullvad's internal services (specifically DNS) are unreachable while the tunnel handshake is otherwise healthy. That was a contributing factor in the 2026-06-04 incident: gateways showed Online while DNS was completely dead.

In-tunnel IPs require Mullvad's internal network to be functioning end-to-end. If those IPs stop responding, the gateway flips Offline and the `VPN_FAILOVER` group reacts.

### Static-route collision avoidance

pfSense installs one static route per Monitor IP, pinning it to that gateway's interface. Two gateways using the **same** Monitor IP would collide: only one route would be installed, the other gateway's monitor traffic would exit the wrong tunnel, marking that gateway falsely Offline.

`100.64.0.31` and `100.64.0.32` are **distinct addresses** in Mullvad's CGNAT space. Each gateway gets a unique IP. No collision.

### Trade-off

Slightly higher latency (~50-60ms vs ~10ms for `1.1.1.1`) and slightly more loss variance (typically 0-3% vs 0%) because the monitor packet now traverses two encrypted hops instead of one. Acceptable: the monitor exists to detect tunnel-internal failure, not to measure latency.

---

## Layer 4: Auto-restart on health-check failure

Direct DNS health probe at the resolver loopback, with an automatic Unbound restart if multiple consecutive probes fail. Implemented as a shell script driven by pfSense's cron package.

### Why cron + script (not Monit)

Monit was the planned tool but is **not present in the pfSense 2.8.x package repo** (deprecated upstream). The cron + script approach achieves the same end behavior with no third-party package, using only built-in pfSense tooling.

### Script

Lives at `/root/unbound_health.sh`. Mode `0755` (executable).

```sh
#!/bin/sh
# Unbound DNS health check + auto-restart
# Restarts Unbound if N consecutive lookups against 127.0.0.1 fail.

STATE=/tmp/unbound_health_fails
LOG=/var/log/unbound_health.log
THRESHOLD=3

if drill -Q example.com @127.0.0.1 > /dev/null 2>&1; then
    echo 0 > "$STATE"
else
    fails=$(cat "$STATE" 2>/dev/null || echo 0)
    fails=$((fails + 1))
    echo "$fails" > "$STATE"
    echo "$(date '+%Y-%m-%d %H:%M:%S') FAIL count=$fails" >> "$LOG"
    if [ "$fails" -ge "$THRESHOLD" ]; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') THRESHOLD reached, restarting Unbound" >> "$LOG"
        /usr/local/sbin/pfSsh.php playback svc restart unbound > /dev/null 2>&1
        echo 0 > "$STATE"
    fi
fi
```

### Behavior

- Probes Unbound with a real DNS query (`drill -Q example.com @127.0.0.1`) against the resolver loopback. Tests end-to-end resolution, not just liveness.
- Resets the failure counter to 0 after each successful query.
- Increments the failure counter and logs each failed query.
- When the failure counter reaches `THRESHOLD` (3), invokes `pfSsh.php playback svc restart unbound` to bounce Unbound through pfSense's service framework (cleaner than `service unbound restart` because it triggers config-aware restart logic).
- Resets the counter to 0 after the restart so a single recovery does not re-trigger.

### Install

1. **Edit File** (`Diagnostics → Edit File`): create `/root/unbound_health.sh`, paste the script body above, Save.
2. **Make executable** (`Diagnostics → Command Prompt`):
   ```
   chmod +x /root/unbound_health.sh
   ```
3. **Sanity test** (`Diagnostics → Command Prompt`):
   ```
   /root/unbound_health.sh && echo "OK exit"
   ```
   Expected: `OK exit`. Any error before that line indicates a problem.

4. **Schedule via Cron package** (`Services → Cron → + Add`):
   - Minute: `*/1`
   - Hour: `*`
   - Day of the Month: `*`
   - Month of the Year: `*`
   - Day of the Week: `*`
   - Who: `root`
   - Command: `/root/unbound_health.sh`
   - Description: `Unbound DNS health check + auto-restart`

5. **Verify after 2-3 minutes** (`Diagnostics → Command Prompt`):
   ```
   ls -la /tmp/unbound_health_fails && cat /tmp/unbound_health_fails
   ```
   Expected: file exists, contents `0`.

### Tuning knobs

- `THRESHOLD` (default 3): consecutive failures before restart. Lower = faster recovery, higher false-positive risk. Higher = slower recovery, more confident trigger.
- Cron interval (`*/1` = every 1 minute): smallest meaningful interval. Combined with threshold 3, this gives a maximum detection-to-recovery window of ~3 minutes.

---

## Verification of the full stack

After applying all four layers:

`Diagnostics → DNS Lookup` → `google.com` → Lookup.

Expected Timings table:

| Name server | Query time |
|---|---|
| `127.0.0.1` | < 100ms |
| `::1` | No response (IPv6 loopback, normal) |
| `100.64.0.31` | 50-70ms |
| `100.64.0.32` | 50-70ms |
| `100.64.0.1` | 50-70ms |
| `100.64.0.2` | 50-70ms |

All four upstream forwarders responding within typical Mullvad tunnel latency.

`Status → Gateways`:

| Gateway | Monitor | Status |
|---|---|---|
| `WAN_DHCP` | (ISP gateway IP) | Online, ~10ms, 0% loss |
| `GW_USA_1` | `100.64.0.31` | Online, 50-60ms, 0-3% loss |
| `GW_USA_2` | `100.64.0.32` | Online, 50-60ms, 0-3% loss |

`Diagnostics → Command Prompt`:

```
cat /tmp/unbound_health_fails
```

Expected: `0` (resets every successful cron run).

`tail -n 20 /var/log/unbound_health.log` (if it exists at all): should be empty or contain only stale entries from before the hardening. Continuous health = no log entries.

---

## Limitations

### Detection-to-recovery latency

The cron-based health check has a 1-minute interval and a 3-failure threshold. The maximum window from "Unbound starts soft-hanging" to "auto-restart fires" is therefore ~3 minutes. Sub-minute recovery would require either a true daemon-style monitor (Monit, not available in 2.8.x) or a custom long-running watchdog process outside cron.

### Script lives outside pfSense XML config

`/root/unbound_health.sh` is a filesystem artifact, not a pfSense GUI-managed object. A pfSense `Diagnostics → Backup & Restore` restore from XML does NOT bring back the script. After any full pfSense restore, re-deploy the script and re-add the cron entry. Document this in any disaster-recovery runbook.

### Dependency on `drill` and `pfSsh.php`

The script calls `drill` (LDNS query tool) and `pfSsh.php` (pfSense's service shell). Both are part of the standard pfSense 2.8.x base image. If a future pfSense release removes either, the script needs an update.

### Filter consistency drift during fallback events

When family-tier forwarders are dead and Unbound falls through to the no-filter or ad-block-only forwarders, queries during that window are not family-content filtered. This is a small and short-lived consistency drift; the trade-off was accepted to keep DNS available during partial outages.

### Mullvad endpoint flapping is undetected as a per-IP event

If a specific Mullvad DNS endpoint (`100.64.0.x`) intermittently drops at the Mullvad side, Unbound's `Keep Probing` and the `TTL for Host Cache Entries = 5 min` mitigate it, but there is no per-forwarder alert. Aggregated symptoms (cache hit rate, query latency) would have to be tracked in a separate monitoring system.

---

## References

- pfSense Unbound docs: https://docs.netgate.com/pfsense/en/latest/services/dns/resolver-config.html
- Mullvad DNS endpoints and content filtering tiers: https://mullvad.net/help/dns-over-https-and-dns-over-tls
- LDNS `drill` manual: https://nlnetlabs.nl/projects/ldns/about/
- Related: [`vpn-failover.md`](vpn-failover.md) for the underlying tunnel and gateway-group configuration that this resilience model assumes.

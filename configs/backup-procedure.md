# Backup & Restore Procedure

Configuration backup strategy for the lab. Covers pfSense, the GS308E switch, and what stays out of git.

---

## What gets backed up

| Source | Format | Where |
|---|---|---|
| pfSense full config | `config-pfsense-YYYY-MM-DD.xml` | Encrypted, offsite + local |
| GS308E switch config | CSV export from `ProSafe Plus` utility | Encrypted, local repo via `age` |
| WireGuard keys | Text (private keys) | Password manager only - never in repo |
| Tailscale auth keys | Text | Password manager only - never in repo |
| Mullvad account number | Text | Password manager only |

The markdown configs in `/configs/` are **documentation**, not a backup - they describe the system, they don't restore it.

---

## pfSense backup

### Manual export (anytime after a change)

1. `Diagnostics > Backup & Restore > Download configuration as XML`
2. Check `Encrypt this configuration file` - use a strong passphrase from the password manager
3. Save as `config-pfsense-YYYY-MM-DD.xml.enc` to the local backups folder
4. Copy to offsite storage (encrypted cloud bucket, external drive, etc.)

### Automatic (recommended)

Install the `AutoConfigBackup` package on pfSense (System > Package Manager). It ships with a free Netgate-hosted tier that keeps encrypted config backups automatically on every change. Alternative: cron + rsync to an S3-compatible bucket.

### Restore

1. Fresh pfSense install on matching version (`pfSense 2.8.1` - check [CHANGELOG.md](../CHANGELOG.md))
2. `Diagnostics > Backup & Restore > Restore configuration`
3. Upload the latest `.xml.enc`, provide passphrase
4. Reboot
5. Re-enter WireGuard private keys from the password manager (they are not in the backup if you exported with the "do not include sensitive data" option - verify beforehand)

---

## GS308E switch backup

The GS308E does not expose config over a standard protocol - it requires the Windows `ProSafe Plus Configuration Utility`.

1. Open the utility, discover the switch
2. `Maintenance > Save Configuration` - saves a binary `.cfg`
3. Also screenshot the VLAN table and port-VLAN membership pages (the binary is opaque; screenshots are the human-readable copy)
4. Encrypt the `.cfg` with `age` or `git-crypt` before committing:

```
age -p -o configs/switch-config-YYYY-MM-DD.cfg.age <raw-cfg-file>
```

### Restore

1. Factory-reset the switch (`Maintenance > Reset`)
2. Upload the `.cfg` via `Maintenance > Restore Configuration`
3. Cross-check against the screenshots and [switch-port-map.md](switch-port-map.md)

---

## What stays out of git

Per `.gitignore`:

- `*.xml` (pfSense configs, even encrypted - belt and suspenders)
- `*.key`, `*.pem` (certs, keys)
- `*.conf` (raw configs)

Encrypted artifacts (`.age`, `.gpg`, `.enc`) are OK to commit **only** if the passphrase is stored separately (password manager, not in the repo, not in commit messages).

---

## Schedule

| When | What |
|---|---|
| Before every pfSense change | Manual XML export |
| Before every switch VLAN change | Manual `.cfg` export + screenshots |
| Monthly | Rotate offsite copy, prune older than 6 months |
| Quarterly | Test restore to a pfSense VM - if it doesn't restore cleanly, the backup is theatre |
| On key rotation | Password manager update, fresh XML export |

---

## Test restore log

Track here when you actually verified a restore works. A backup you haven't restored is a backup you don't have.

| Date | Source file | Target | Result |
|---|---|---|---|
| _pending_ | _pending_ | _pending_ | _pending_ |

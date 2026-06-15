# MusterBank AG — Config Backup Playbooks

## Problem Solved
EVE-NG Pro bug (observed after upgrade): NX-OS nodes lose `startup-config` on
lab restart. These playbooks address it with a **dual approach**:

1. **On-device save** — `copy running-config startup-config` (NX-OS) or
   `write memory` (IOS) so the device itself has the latest config persisted.
2. **Off-device backup** — copies the running-config to the AWX/Ansible server
   at `/opt/backups/musterbank/` with a timestamp, plus a `latest.cfg` symlink.

---

## Files

| File | Purpose |
|------|---------|
| `backup_all.yml` | Master playbook — runs both phases |
| `backup_nxos.yml` | NX-OS only (Spines + Leafs, both DCs) |
| `backup_ios_servers.yml` | IOS servers only (both DCs) |
| `dc1/inventory/hosts.ini` | DC1 Frankfurt inventory |
| `dc2/inventory/hosts.ini` | DC2 Nürnberg inventory |

---

## Backup Directory Structure (on AWX server)

```
/opt/backups/musterbank/
├── dc1_frankfurt/
│   ├── F-SPINE-01/
│   │   ├── 20260615_183000.cfg
│   │   └── latest.cfg -> 20260615_183000.cfg
│   ├── F-SPINE-02/
│   ├── F-LEAF-01/ ... F-LEAF-05/
│   └── servers/
│       ├── SRV1/ ... SRV3/
│       └── SRV11/ ... SRV13/
└── dc2_nurnberg/
    ├── N-SPINE-01/
    ├── N-SPINE-02/
    ├── N-LEAF-01/ ... N-LEAF-05/
    └── servers/
        ├── SRV21/ ... SRV23/
        └── SRV31/ ... SRV33/
```

---

## AWX Setup

### Option A — Two Job Templates (recommended, matches your existing DC1/DC2 split)

| Template Name | Playbook | Inventory |
|---------------|----------|-----------|
| `MusterBank-Backup-DC1` | `backup_all.yml` | DC1-Fabric |
| `MusterBank-Backup-DC2` | `backup_all.yml` | DC2-Fabric |

### Option B — One Job Template (combined inventory)
Create a combined AWX inventory with all devices and run `backup_all.yml` once.

### Scheduling
Add an AWX Schedule to each template to run automatically — e.g., every hour
or before any planned lab restart.

---

## Quick Restore

To restore a device from backup after EVE-NG wipes the startup-config:

```bash
# On the AWX server — paste the backup into the device console or via SSH:
cat /opt/backups/musterbank/dc1_frankfurt/F-LEAF-01/latest.cfg

# Or push via Ansible ad-hoc:
ansible F-LEAF-01 -i dc1/inventory/hosts.ini -m cisco.nxos.nxos_config \
  -a "src=/opt/backups/musterbank/dc1_frankfurt/F-LEAF-01/latest.cfg"
```

---

## Notes
- NX-OS SSH uses legacy KEX algorithms — `ansible_ssh_common_args` in inventory
  handles this (same pattern as your existing lab playbooks).
- IOS servers connect via VRF Management (Ethernet0/3) — Ansible reaches them
  via their mgmt IPs (.80–.91) which are in the management VRF.
- The `latest.cfg` symlink always points to the most recent successful backup,
  making restore scripts simple.

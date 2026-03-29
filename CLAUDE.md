# CLAUDE.md

## Project overview

Proxmox-HA-Shutdown-Helper provides POSIX shell scripts, systemd units,
and NUT configuration to gracefully shut down a Proxmox VE HA cluster
during a UPS-triggered forced shutdown (FSD) and automatically restore
HA after power returns.

## Key files

- `pve-nut-shutdown` — main shutdown script
- `pve-nut-shutdown-restore` — post-boot HA restore script
- `pve-nut-shutdown-zfsfix` — ZFS replication repair script
- `pve-nut-shutdown.service` — systemd unit for shutdown
- `pve-nut-shutdown-restore.service` — systemd unit for restore
- `pve-nut-shutdown-zfsfix.service` — systemd unit for ZFS repair
- `nut/upsmon.conf.example` — example NUT client configuration
- `AGENTS.md` — contributor/agent coding guidelines
- `README.md` — user-facing documentation

## Coding standards

- Shell scripts must be **POSIX compliant** (`#!/bin/sh`, no bashisms).
- Use **4 spaces** for indentation.
- Markdown files: wrap lines at **80 characters**.

## Pre-commit checks

Run these before committing:

```sh
bash -n pve-nut-shutdown pve-nut-shutdown-restore pve-nut-shutdown-zfsfix
shellcheck pve-nut-shutdown pve-nut-shutdown-restore pve-nut-shutdown-zfsfix
markdownlint README.md AGENTS.md
```

Install missing tools:

```sh
apt-get install -y shellcheck
npm install -g markdownlint-cli
```

## How the shutdown script works

1. Parses `--maintenance` flag (optional).
2. Guards against running twice (`systemctl` check + `flock`).
3. Enables HA node-maintenance mode to freeze the scheduler.
4. Stops HA daemons (`pve-ha-crm`, `pve-ha-lrm`, `watchdog-mux`).
5. Waits for CRM to write status (configurable).
6. Shuts down all VMs (`qm`) and containers (`pct`) with logging.
7. Waits for guest processes to exit (configurable timeout).
8. Powers off the node.

## How the restore script works

1. Waits for the cluster to become reachable (up to 10 min).
2. Disables HA maintenance mode on the local node.

## How the ZFS repair script works

1. Waits for `pvesr` to be available.
2. Finds replication jobs in an error state.
3. Removes stale `__replicate_` snapshots for those jobs.
4. Triggers immediate full re-sync via `pvesr schedule-now`.

## Boot-time service chain

```
pve-cluster → corosync → pve-ha-crm
                            ↓
              pve-nut-shutdown-restore  (wait for quorum, disable maintenance)
                            ↓
              pve-nut-shutdown-zfsfix   (fix broken replication jobs)
```

## Design decisions

- All cluster VMs/CTs are shut down intentionally (not just local
  ones) because the three-node cluster shares power infrastructure.
- All timeouts are configurable via environment variables.

# Proxmox-HA-Shutdown-Helper

This repository contains scripts and systemd units that allow each node
in a Proxmox HA cluster to shut down gracefully during a power outage
and restore itself automatically when power returns.  It is typically
triggered by [NUT](https://networkupstools.org/) when a forced shutdown
(FSD) occurs.

## Repository layout

| File | Purpose |
| ---- | ------- |
| `pve-nut-shutdown` | Main shutdown script. |
| `pve-nut-shutdown-restore` | Post-boot restore script. |
| `pve-nut-shutdown.service` | Systemd unit for the shutdown script. |
| `pve-nut-shutdown-restore.service` | Systemd unit for the restore script. |
| `pve-nut-shutdown-zfsfix` | ZFS replication repair script. |
| `pve-nut-shutdown-zfsfix.service` | Systemd unit for the ZFS repair script. |
| `nut/upsmon.conf.example` | Example NUT client configuration. |

## Installation

On **every** node in the cluster:

```bash
# Copy scripts
cp pve-nut-shutdown /usr/local/sbin/pve-nut-shutdown
cp pve-nut-shutdown-restore /usr/local/sbin/pve-nut-shutdown-restore
cp pve-nut-shutdown-zfsfix /usr/local/sbin/pve-nut-shutdown-zfsfix
chmod 755 /usr/local/sbin/pve-nut-shutdown \
          /usr/local/sbin/pve-nut-shutdown-restore \
          /usr/local/sbin/pve-nut-shutdown-zfsfix

# Install systemd units
cp pve-nut-shutdown.service /etc/systemd/system/
cp pve-nut-shutdown-restore.service /etc/systemd/system/
cp pve-nut-shutdown-zfsfix.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable pve-nut-shutdown-restore.service \
                 pve-nut-shutdown-zfsfix.service
```

Then configure NUT to call the shutdown script when FSD fires.  See
`nut/upsmon.conf.example` for a ready-to-adapt example.

The scripts are self-contained — they do not rely on SSH or extra
packages, so they can run even if the cluster is losing quorum.

## NUT integration

Copy `nut/upsmon.conf.example` to `/etc/nut/upsmon.conf` and edit the
`MONITOR` line to match your UPS name, host, and credentials.  The
key settings are:

```
NOTIFYCMD /usr/local/sbin/pve-nut-shutdown
NOTIFYFLAG FSD  SYSLOG+EXEC
```

This tells NUT to execute the shutdown script only on FSD events while
logging all other events normally.

## Planned maintenance

To drain a node for planned work, run the script manually with
`--maintenance`:

```bash
pve-nut-shutdown --maintenance
```

The script will place the node into maintenance mode and stop guests
gracefully.  NUT continues to call the script without arguments on
FSD events.

## What the shutdown script does

| Step | Purpose |
| ---- | ------- |
| Check node status | Exit if system is already stopping. |
| Acquire lock | Prevent duplicate runs via `flock`. |
| Enable node maintenance | Freeze HA scheduler to prevent migrations/fencing. |
| Stop HA daemons | Prevent services from restarting during shutdown. |
| Settle CRM | Give the CRM time to write status. |
| Shut down guests | Stop each VM (`qm`) and container (`pct`) gracefully. |
| Wait for guests | Wait for VMs/CTs to stop cleanly. |
| Power off node | Halt the host once all guests are stopped. |

Because the script runs on every node, there is no designated "master";
each node looks after itself.

## Post-power-restore

The `pve-nut-shutdown-restore` service runs automatically at boot.  It
waits for cluster quorum (up to `CLUSTER_TIMEOUT` seconds, default 600)
and then disables maintenance mode so the HA scheduler can manage guests
on the node again.

In a 3-node cluster, quorum requires at least 2 nodes.  The first node
to boot will wait in the restore script until a second node comes up.
The 10-minute default timeout covers most staggered boot scenarios.
If the timeout expires, systemd retries the service up to 3 times
(every 30 seconds).

If all retries fail, run the restore manually once quorum is available:

```bash
pve-nut-shutdown-restore
```

## ZFS replication repair

After a power event, Proxmox ZFS replication jobs often break because
the `__replicate_` snapshots on source and target nodes are out of
sync.  Normally this requires manually deleting and recreating the
replication job from the web UI.

The `pve-nut-shutdown-zfsfix` service automates this.  It runs after
the restore service and:

1. Checks `pvesr status` for jobs in an error state.
2. Removes stale `__replicate_` snapshots for those jobs.
3. Triggers an immediate full re-sync via `pvesr schedule-now`.

The service retries up to 3 times if `pvesr` is not yet available.
To run it manually:

```bash
pve-nut-shutdown-zfsfix
```

## Configuration

All timeouts can be tuned via environment variables.  Defaults are
shown below:

**Shutdown script** (`pve-nut-shutdown`):

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `GUEST_TIMEOUT` | `60` | Per-guest shutdown timeout (seconds). |
| `WAIT_TIMEOUT` | `300` | Max time to wait for all guests to stop. |
| `HA_TIMEOUT` | `15` | Timeout for `ha-manager` maintenance command. |
| `CRM_SETTLE` | `8` | Seconds to let the CRM write its status. |

**Restore script** (`pve-nut-shutdown-restore`):

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `CLUSTER_TIMEOUT` | `600` | Max time to wait for cluster quorum. |
| `RETRY_INTERVAL` | `10` | Seconds between quorum checks. |

**ZFS repair script** (`pve-nut-shutdown-zfsfix`):

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `REPL_TIMEOUT` | `120` | Max wait for `pvesr` to be available. |
| `RETRY_INTERVAL` | `10` | Seconds between availability checks. |

To override via systemd, create a drop-in:

```bash
systemctl edit pve-nut-shutdown.service
```

```ini
[Service]
Environment=GUEST_TIMEOUT=120
Environment=WAIT_TIMEOUT=600
```

Or pass them inline for a manual run:

```bash
GUEST_TIMEOUT=120 WAIT_TIMEOUT=600 pve-nut-shutdown
```

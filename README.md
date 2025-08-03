# Proxmox-HA-Shutdown-Helper

This repository contains `pve-nut-shutdown`, a small helper script that allows
each node in a Proxmox HA cluster to shut down gracefully during a power
outage. It is typically triggered by
[NUT](https://networkupstools.org/) when a forced shutdown (FSD) occurs.

## Installation

1. Copy `pve-nut-shutdown` to `/usr/local/sbin/pve-nut-shutdown` on every node.
2. Make it executable: `chmod 755 /usr/local/sbin/pve-nut-shutdown`.
3. Configure NUT to call the script when FSD fires.

The script is self-contained—it does not rely on SSH or extra packages, so it
can run even if the cluster is losing quorum.

## What the script does

| Step | Purpose |
| ---- | ------- |
| Enable node maintenance | Freeze HA scheduler to prevent migrations/fencing. |
| Stop HA daemons | Prevent services from restarting during shutdown. |
| Sleep 8 seconds | Give the CRM time to write status. |
| Shut down guests | Stop each VM (`qm`) and container (`pct`) gracefully. |
| Wait for guests | Allow up to five minutes for VMs/CTs to stop cleanly. |
| Power off node | Halt the host once all guests are stopped. |

Because the script runs on every node, there is no designated “master”; each
node looks after itself.

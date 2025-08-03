# Proxmox-HA-Shutdown-Helper
Helps to shutdown a simple Proxmox HA cluster primarily used with NUT for power outages.


Create /usr/local/sbin/pve-nut-shutdown on every node and make it executable (chmod 755 …).
The script below is deliberately self-contained—no extra packages and no SSH to other nodes, so it can still run even if the cluster is already losing quorum.

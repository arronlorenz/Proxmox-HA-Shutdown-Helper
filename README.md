# Proxmox-HA-Shutdown-Helper
Helps to shutdown a simple Proxmox HA cluster primarily used with NUT for power outages.


Create /usr/local/sbin/pve-nut-shutdown on every node and make it executable (chmod 755 …).
The script below is deliberately self-contained—no extra packages and no SSH to other nodes, so it can still run even if the cluster is already losing quorum.


What it does

Step	Why it matters
node-maintenance enable	Tells the HA CRM to freeze this node; nothing will be migrated off or fenced while we are shutting down. 
pve.proxmox.com
systemctl stop pve-ha-crm pve-ha-lrm	Official Proxmox advice before a full-cluster power-off. 
forum.proxmox.com
Guest loops (qm / pct)	Gracefully power down every VM/CT still running. 
forum.proxmox.com

Because the script runs on every node, you don’t have to decide which machine is the “master”. Each node looks after itself.

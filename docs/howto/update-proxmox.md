# Update Proxmox VE

Apply available package updates to the Proxmox host (the NUC) through the web UI.

## Prerequisites

- Access to the Proxmox web UI: [https://192.168.2.210:8006](https://192.168.2.210:8006)
- Login credentials for the host
- A moment where a reboot is acceptable (VMs and containers go down during restart)

## Steps

1. Open the web UI at [https://192.168.2.210:8006](https://192.168.2.210:8006) and log in.
2. In the left tree, expand **Datacenter** and select the node **pve**.
3. In the node menu, click **Updates**.
4. Click **Upgrade**. A console opens and installs the available packages.
5. When the upgrade finishes, reboot the host (**More → Reboot**, or run `reboot` in
   the console).

## Verify

- After the host comes back up, log in again and open **pve → Updates**; the list
  should show no pending packages.
- Confirm the version under **pve → Summary** (or run `pveversion` in a shell).

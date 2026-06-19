# Add UNAS NFS Storage to Proxmox

Attach a UniFi UNAS NFS export to the Proxmox host (the NUC) as a storage target
for backups and templates.

This covers Proxmox-level storage (vzdump backups, ISOs, container templates).
Mounting an NFS export *inside* a container for data access is a separate step —
host-mount plus bind-mount — handled per guest, not here.

## Prerequisites

- Access to the Proxmox web UI: [https://192.168.2.210:8006](https://192.168.2.210:8006)
- A Shared Drive on the [UNAS 2](../hardware/unas.md) with its NFS export enabled,
  scoped to the LAN subnet `192.168.2.0/24`
- The UNAS reachable at `192.168.2.24` (see the [network overview](../network/overview.md))

## Steps

1. Open the web UI and select **Datacenter** in the left tree.
2. Go to **Storage → Add → NFS**.
3. Fill in the dialog:
   - **ID**: a short name for the mount, e.g. `unas-eve`.
   - **Server**: `192.168.2.24`.
   - **Export**: click the field and pick the detected export from the dropdown
     (Proxmox scans the server). If the list is empty, the export is not reachable
     — check that NFS is enabled and the subnet scope includes `192.168.2.0/24`.
   - **Content**: select **VZDump backup file** (add **ISO image** and **Container
     template** if this drive should also hold those).
   - **Nodes**: restrict to **pve** (the only node).
4. Click **Add**. The storage appears under the node in the left tree.

## Verify

- The new storage shows online (green) under **pve** in the left tree.
- Open **pve → \<storage\> → Summary**; usage and capacity report correctly.
- Run a test write: back up a small container or VM to it via
  **\<guest\> → Backup → Backup now**, target the new storage, and confirm it
  completes and the file is listed under **\<storage\> → Backups**.

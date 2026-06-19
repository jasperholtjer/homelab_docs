# Add UNAS NFS Storage to Proxmox

Attach a UniFi UNAS NFS export to the Proxmox host (the NUC) as a storage target
for backups and templates.

This covers Proxmox-level storage (vzdump backups, ISOs, container templates).
Mounting an NFS export *inside* a container for data access is a separate step —
host-mount plus bind-mount — handled per guest, not here.

## Prerequisites

- Access to the Proxmox web UI: [https://192.168.2.210:8006](https://192.168.2.210:8006)
- Access to the [UNAS 2](../hardware/unas.md) web UI, reachable at `192.168.2.24`
  (see the [network overview](../network/overview.md))
- A Shared Drive on the UNAS (e.g. `EVE`)

## Grant the Proxmox host access on the UNAS

The UNAS controls NFS access with a trusted-client list, not a Linux-style
`/etc/exports` ACL. The Proxmox host must be listed there or the mount fails with
`access denied by server`.

1. On the UNAS, go to **Settings → File Services → NFS** and enable NFS.
2. Click **Add NFS Connections**, enter the Proxmox host IP `192.168.2.210` as the
   trusted **Hostname or IP**, and **Add** it. Leave **NFS Write Mode** on `async`
   (faster; the data is idempotent and re-derivable) unless backup crash-safety
   demands `sync`.
3. Under **Shared Drives Permissions → Add Shared Drives**, attach the drive (`EVE`).

## Add the storage in Proxmox

1. Open the web UI and select **Datacenter** in the left tree.
2. Go to **Storage → Add → NFS**.
3. Fill in the dialog:
   - **ID**: a short name for the mount, e.g. `unas-eve`.
   - **Server**: `192.168.2.24`.
   - **Export**: pick the detected export from the dropdown (Proxmox scans the
     server). If the list is empty, type the path manually — it is the share name
     as an absolute path (e.g. `/EVE`), not bare `EVE`. Confirm it from a Linux
     client with `showmount -e 192.168.2.24` if unsure.
   - **Content**: select **Backup** (add **ISO image** and **Container template**
     if this drive should also hold those).
   - **Nodes**: restrict to **pve** (the only node).
4. Click **Add**. The storage appears under the node in the left tree.

## Verify

- The new storage shows online (green) under **pve** in the left tree.
- Open **pve → \<storage\> → Summary**; usage and capacity report correctly.
- Run a test write: back up a small container or VM to it via
  **\<guest\> → Backup → Backup now**, target the new storage, and confirm it
  completes and the file is listed under **\<storage\> → Backups**.

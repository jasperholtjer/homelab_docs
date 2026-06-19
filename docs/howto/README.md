# How-To Guides

Task-oriented runbooks for things that are done rarely enough to forget. Each guide
is one file named after the task (kebab-case), e.g. `add-proxmox-vm.md`.

## Guides

- [Update Proxmox VE](update-proxmox.md) — apply package updates via the web UI
- [Reserve an IP Address in the KPN Box](reserve-ip-kpn-box.md) — bind a device to a fixed IP via DHCP reservation
- [Add UNAS NFS Storage to Proxmox](add-nfs-storage-proxmox.md) — attach a UNAS NFS export as a Proxmox storage target

## Writing a Guide

- Start with the goal in one sentence: what does this guide accomplish?
- List prerequisites (access, IPs, credentials needed).
- Number the steps; keep each step a single action.
- Use fenced code blocks with a language tag for commands.
- End with a verification step: how to confirm it worked.

# Deploy the Dagster Orchestration LXC

Stand up the LXC on the NUC that runs the Dagster orchestrator and the `corpus`
binary for the [eve-industry-corpus](../projects/eve-industry-corpus.md)
platform — phase 3 of that project's build order.

This is the producer's control plane: it mounts the UNAS over NFS, drives the
`corpus` subcommands, and keeps Dagster's run-state on the NUC's NVMe. The
orchestration code (Dagster assets, systemd units, deploy scripts) lives in its
own `eve-corpus-orchestration` repository and is pulled in during the runtime
step — this guide covers the homelab-side host setup, not the application code.

## Prerequisites

- Access to the Proxmox host shell — **pve → Shell** in the web UI, or
  `ssh root@192.168.2.210` (see the [NUC](../hardware/nuc.md))
- The static IP reserved for the Dagster-LXC: `192.168.2.211` (the
  [network overview](../network/overview.md) is the source of truth for addresses)
- Phases 1–2 done: the UNAS `EVE` Shared Drive exports NFS and the Proxmox host
  is a trusted NFS client (see
  [Add UNAS NFS Storage to Proxmox](add-nfs-storage-proxmox.md))

All `pct` commands run on the Proxmox host, not inside the container.

## 1. Create the LXC

Use an **unprivileged** container on **Debian 13** with the rootfs on the NVMe.
Unprivileged is the safer default; the NFS write-permission quirk it introduces
is handled in the next step with a UID/GID map.

1. Fetch the template if it is not already present (**pve → local → CT
   Templates**, or on the host):

   ```bash
   pveam update
   pveam download local debian-13-standard_13.1-2_amd64.tar.zst
   ```

2. Create the container via **Create CT**, with these non-default settings:

   - **General**: CT ID `211` (matches the IP octet), Hostname `dagster`,
     **Unprivileged container** enabled. Set an SSH key and/or root password —
     both are optional, since `pct enter 211` from the host needs neither.
   - **Template**: storage `local`, the `debian-13-standard` image.
   - **Disks**: rootfs on `local-lvm` (the NVMe), `20` GiB — room for the venv,
     the `corpus` binary, the SQLite `DAGSTER_HOME`, and logs.
   - **CPU**: `2` cores (the `corpus` binary does the heavy work; Dagster is light).
   - **Memory**: `4096` MiB RAM, `512` MiB swap — the top of the project's
     capacity budget for this guest.
   - **Network**: bridge `vmbr0`, IPv4 **Static** `192.168.2.211/24`, gateway
     `192.168.2.254`.

   Do not start it yet — the next setting and the NFS map are applied while it
   is stopped.

3. Enable the `nesting` feature. Debian 13 ships systemd 257, which needs
   `nesting=1` to access the cgroup and `/sys` mounts it expects inside an
   unprivileged container — without it the Dagster systemd units misbehave. This
   is not about running containers inside the container.

   ```bash
   pct set 211 -features nesting=1
   ```

### Verify

Start the container and confirm systemd, the address, and LAN reachability:

```bash
pct start 211
pct exec 211 -- systemctl is-system-running    # running (or degraded — check systemctl --failed)
pct exec 211 -- ip -4 addr show eth0            # 192.168.2.211/24
pct exec 211 -- ping -c1 192.168.2.254          # gateway
pct exec 211 -- ping -c1 192.168.2.24           # UNAS, same 2.5G switch
```

## 2. Map the container to the NFS owner (UID/GID 988)

The UNAS writes Shared-Drive files as its internal `unifi-core` account — group
ID **988**. An unprivileged container shifts every ID by +100000, so by default
the container's writes arrive as the wrong owner and the NFS server rejects them.
Map the single ID 988 straight through, leaving the safe +100000 shift intact for
everything else.

> [!NOTE]
> 988 is the value UniFi OS uses today; it is firmware-dependent, not
> contractual. Confirm it against the real export with `ls -n` once the share is
> mounted (next section) and substitute the actual group ID if it differs.

1. Allow root to delegate ID 988. Append to both `/etc/subuid` and `/etc/subgid`
   on the host:

   ```text
   root:988:1
   ```

2. Add the mapping to `/etc/pve/lxc/211.conf`:

   ```text
   lxc.idmap: u 0 100000 988
   lxc.idmap: u 988 988 1
   lxc.idmap: u 989 100989 64547
   lxc.idmap: g 0 100000 988
   lxc.idmap: g 988 988 1
   lxc.idmap: g 989 100989 64547
   ```

   Each line reads `<container-ID> <host-ID> <count>`. The three blocks cover the
   full 0–65535 range with no gap or overlap: IDs below 988 keep the +100000
   shift, 988 maps one-to-one, and 989 upward shift again (`65536 − 989 = 64547`).

### Verify

```bash
pct reboot 211
pct status 211        # status: running
```

A container that fails to start means the `root:988:1` line is missing from
`/etc/subuid` or `/etc/subgid`. The write-path proof comes in the next section,
once the share is mounted.

## 3. Mount the UNAS NFS and bind-mount it into the container

An unprivileged container cannot mount NFS itself, so the host mounts the export
and hands it to the container as a bind-mount. This is separate from the
Proxmox-level storage in
[Add UNAS NFS Storage to Proxmox](add-nfs-storage-proxmox.md), which serves
backups, not data access.

1. Find the real export path. UniFi OS does not export the bare share name; it
   exports a UUID path under the drive. Ask the server:

   ```bash
   showmount -e 192.168.2.24
   ```

   The path looks like `/volume/<drive-uuid>/.srv/.unifi-drive/EVE/.data` — copy
   it verbatim.

2. Add the host mount to `/etc/fstab`. Use **NFSv3**: `showmount` reports the v3
   export path, and UniFi OS does not serve that path over NFSv4 (a v4 mount
   fails with `No such file or directory`).

   ```text
   192.168.2.24:/volume/<drive-uuid>/.srv/.unifi-drive/EVE/.data  /mnt/eve  nfs  defaults,_netdev,vers=3  0  0
   ```

   `_netdev` defers the mount until the network is up; `vers=3` pins the working
   protocol.

3. Mount it. A changed `fstab` is not picked up until systemd reloads:

   ```bash
   mkdir -p /mnt/eve
   systemctl daemon-reload
   mount /mnt/eve
   ls -n /mnt/eve        # entries show group 988
   ```

4. Bind-mount the host path into the container:

   ```bash
   pct set 211 -mp0 /mnt/eve,mp=/mnt/eve
   pct reboot 211
   ```

5. Create a service account in group 988 inside the container. Container root
   maps to host 100000 — neither owner nor group 988 — so it lands in `other` and
   cannot write. Work runs as a user that *is* in group 988.

   ```bash
   pct enter 211
   groupadd -g 988 nfsdata
   useradd -m -u 1000 -g 988 -s /bin/bash corpus
   ```

   The `corpus` account owns the venv, the binary, and `DAGSTER_HOME`. Its primary
   group 988 stamps everything it writes to the share with the group the UNAS can
   read back.

### Verify

Write and delete a probe as `corpus`:

```bash
su - corpus -c 'touch /mnt/eve/.probe && ls -n /mnt/eve/.probe'   # creates file, group 988
su - corpus -c 'rm /mnt/eve/.probe'                                # delete succeeds
```

The file's owner displays as `65534` (nobody): the container's UID has no name on
the UNAS, which is cosmetic on read-back. Write access hinges on **group 988** and
the group-writable parent directory, so create, replace, and delete all work —
sufficient for the write-once `parquet + _INDEX.json + _DONE` contract.

# Deploy the MLflow LXC

Stand up the LXC on the NUC that runs the MLflow tracking + registry server and
its Postgres backing store for the
[eve-industry-predict](../projects/eve-industry-predict.md) ML platform — phase 2
of that project's build order.

This is the training tier's always-on control plane: the workstation logs runs
and pushes pyfunc artefacts to it, and consumers resolve model versions from it.
The server proxies artefacts to a dedicated UNAS Shared Drive (`ML`) over NFS, so
clients need only HTTP. The training and feature code lives in the
`eve-industry-predict` repository — this guide covers the homelab-side host
setup, not the application code.

## Prerequisites

- Access to the Proxmox host shell — **pve → Shell** in the web UI, or
  `ssh root@192.168.2.210` (see the [NUC](../hardware/nuc.md))
- The static IP reserved for the MLflow-LXC: `192.168.2.213` (the
  [network overview](../network/overview.md) is the source of truth for
  addresses; reserve it per [Reserve an IP Address in the KPN Box](reserve-ip-kpn-box.md))
- A dedicated UNAS Shared Drive `ML` with NFS enabled, trusting the host
  `192.168.2.210` (phase 1; same UniFi steps as the `EVE` share in
  [Add UNAS NFS Storage to Proxmox](add-nfs-storage-proxmox.md), pointed at a new
  drive). Keep model-derived files off the `EVE` medallion share.

All `pct` commands run on the Proxmox host, not inside the container.

## 1. Create the LXC

Use an **unprivileged** container on **Debian 13** with the rootfs on the NVMe.
Unprivileged is the safer default; the NFS write-permission quirk it introduces
is handled in step 2 with a UID/GID map — identical to the Dagster LXC.

1. Fetch the template if it is not already present (**pve → local → CT
   Templates**, or on the host):

   ```bash
   pveam update
   pveam download local debian-13-standard_13.1-2_amd64.tar.zst
   ```

2. Create the container via **Create CT**, with these non-default settings:

   - **General**: CT ID `213` (matches the IP octet), Hostname `mlflow`,
     **Unprivileged container** enabled. An SSH key/root password is optional —
     `pct enter 213` from the host needs neither.
   - **Template**: storage `local`, the `debian-13-standard` image.
   - **Disks**: rootfs on `local-lvm` (the NVMe), `16` GiB — room for the venv
     and the Postgres metadata store. Artefacts live on the NAS, not here.
   - **CPU**: `2` cores.
   - **Memory**: `2048` MiB RAM, `512` MiB swap — the charter budget (MLflow
     ~300 MB + Postgres ~500 MB + OS).
   - **Network**: bridge `vmbr0`, IPv4 **Static** `192.168.2.213/24`, gateway
     `192.168.2.254`.

   Do not start it yet — the next setting and the NFS map are applied while it is
   stopped.

3. Enable the `nesting` feature. Debian 13 ships systemd 257, which needs
   `nesting=1` to access the cgroup and `/sys` mounts it expects inside an
   unprivileged container — without it the systemd units misbehave.

   ```bash
   pct set 213 -features nesting=1
   ```

### Verify

```bash
pct start 213
pct exec 213 -- systemctl is-system-running    # running (or degraded — check systemctl --failed)
pct exec 213 -- ip -4 addr show eth0            # 192.168.2.213/24
pct exec 213 -- ping -c1 192.168.2.254          # gateway
pct exec 213 -- ping -c1 192.168.2.24           # UNAS, same 2.5G switch
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

2. Add the mapping to `/etc/pve/lxc/213.conf`:

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
   shift, 988 maps one-to-one, and 989 upward shifts again (`65536 − 989 =
   64547`).

### Verify

```bash
pct reboot 213
pct status 213        # status: running
```

A container that fails to start means the `root:988:1` line is missing from
`/etc/subuid` or `/etc/subgid`.

## 3. Mount the `ML` share and bind-mount it into the container

An unprivileged container cannot mount NFS itself, so the host mounts the export
and hands it to the container as a bind-mount.

1. Find the real export path. UniFi OS exports a UUID path under the drive, not
   the bare share name:

   ```bash
   showmount -e 192.168.2.24
   ```

   The path looks like `/volume/<drive-uuid>/.srv/.unifi-drive/ML/.data` — copy
   it verbatim.

2. Add the host mount to `/etc/fstab`. Use **NFSv3**: `showmount` reports the v3
   export path, and UniFi OS does not serve it over NFSv4.

   ```text
   192.168.2.24:/volume/<drive-uuid>/.srv/.unifi-drive/ML/.data  /mnt/ml  nfs  defaults,_netdev,vers=3  0  0
   ```

3. Mount it and create the artefact directory:

   ```bash
   mkdir -p /mnt/ml
   systemctl daemon-reload
   mount /mnt/ml
   ls -n /mnt/ml         # entries show group 988
   ```

4. Bind-mount the host path into the container:

   ```bash
   pct set 213 -mp0 /mnt/ml,mp=/mnt/ml
   pct reboot 213
   ```

5. Create a service account in group 988 inside the container. Container root
   maps to host 100000 — neither owner nor group 988 — so it lands in `other` and
   cannot write. MLflow runs as a user that *is* in group 988.

   ```bash
   pct enter 213
   groupadd -g 988 nfsdata
   useradd -m -u 1000 -g 988 -s /bin/bash mlflow
   install -d -o mlflow -g 988 /mnt/ml/mlflow-artifacts
   ```

### Verify

Write and delete a probe as `mlflow`:

```bash
su - mlflow -c 'touch /mnt/ml/mlflow-artifacts/.probe && ls -n /mnt/ml/mlflow-artifacts/.probe'   # group 988
su - mlflow -c 'rm /mnt/ml/mlflow-artifacts/.probe'                                                 # delete succeeds
```

The file's owner displays as `65534` (nobody) — cosmetic on read-back; write
access hinges on **group 988** and the group-writable parent directory.

## 4. Install the Postgres backing store

MLflow keeps its experiment and registry metadata in Postgres, on `localhost`
only (the server is co-located). Debian 13 ships Postgres 17.

```bash
apt-get update && apt-get install -y postgresql
sudo -u postgres psql -c "CREATE ROLE mlflow LOGIN PASSWORD 'CHANGE_ME';"
sudo -u postgres createdb -O mlflow mlflow
sudo -u postgres psql -d mlflow -c '\conninfo'
```

Record the password — it goes in the systemd unit's backend-store URI (step 5).

## 5. Install and run the MLflow server

MLflow runs as the `mlflow` service account (group 988, so it can write artefacts
to the NAS). Install it into a user-local venv with `uv`, pinning the major
version to the predict client (`mlflow>=2.18` in the predict `pyproject.toml`) so
server and client stay compatible.

1. Install `uv` and MLflow as `mlflow`:

   ```bash
   su - mlflow
   curl -LsSf https://astral.sh/uv/install.sh | sh
   source ~/.bashrc
   uv venv ~/mlflow-venv
   uv pip install --python ~/mlflow-venv 'mlflow>=2.18' 'psycopg2-binary'
   ~/mlflow-venv/bin/mlflow --version
   exit
   ```

2. Install the systemd unit (as root). The server binds all interfaces so the
   workstation can reach it, uses Postgres as the backend, and **proxies
   artefacts** to the NFS-mounted `ML` share:

   ```bash
   cat > /etc/systemd/system/mlflow.service <<'EOF'
   [Unit]
   Description=MLflow tracking + registry server
   After=network-online.target postgresql.service
   Wants=network-online.target
   Requires=postgresql.service

   [Service]
   User=mlflow
   Group=988
   ExecStart=/home/mlflow/mlflow-venv/bin/mlflow server \
     --backend-store-uri postgresql://mlflow:CHANGE_ME@localhost/mlflow \
     --artifacts-destination file:/mnt/ml/mlflow-artifacts \
     --serve-artifacts \
     --host 0.0.0.0 --port 5000
   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   EOF

   systemctl daemon-reload
   systemctl enable --now mlflow
   ```

### Verify

Confirm the service, the schema (MLflow runs its own migrations on first start),
and reachability from the container:

```bash
systemctl status mlflow                              # active (running)
pct exec 213 -- curl -sf http://localhost:5000/health   # OK
sudo -u postgres psql -d mlflow -c '\dt' | head        # experiments, runs, registered_models, ...
```

Then, **from the workstation**, drive the full chain — log a run and an artefact
through the HTTP proxy:

```powershell
$env:MLFLOW_TRACKING_URI = "http://192.168.2.213:5000"
uv run python -c "import mlflow; mlflow.set_experiment('smoke'); mlflow.start_run(); mlflow.log_metric('x', 1); mlflow.log_text('hello', 'probe.txt'); mlflow.end_run(); print('logged')"
```

The run appears in the UI at [192.168.2.213:5000](http://192.168.2.213:5000), and
`probe.txt` lands under `/mnt/ml/mlflow-artifacts/` on the NAS — confirming the
tracking DB, the artefact proxy, and the NFS write path all work end to end.

> [!NOTE]
> No authentication is enabled — the server is LAN-only on a trusted `/24`. If
> the platform later needs auth, MLflow's built-in basic-auth
> (`--app-name basic-auth`) is the lightest option; add it before exposing the
> port beyond the LAN.

## 6. Backups

`vzdump` the LXC to the UNAS backup storage on the same schedule as the other
guests. The Postgres metadata lives on the NVMe; the dump is its recovery point.
The artefacts on the `ML` share are backed up with that drive, not the container.

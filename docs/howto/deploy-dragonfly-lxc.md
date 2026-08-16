# Deploy the Dragonfly LXC

Stand up the LXC on the NUC that runs Dragonfly as the Feast **online store** for
the [eve-industry-predict](../projects/eve-industry-predict.md) ML platform —
phase 3 of that project's build order.

Dragonfly is a Redis-compatible, in-memory feature cache. Inference-time
consumers read point-in-time feature vectors from it; `feast materialize` jobs
write them. It is self-contained — no NFS, no shared storage — so this is the
simplest of the platform guests.

> [!NOTE]
> Nothing reads this store yet. It is provisioned ahead of the predict charter's
> "after Q5" trigger (open Q 7.6); until `feast materialize` runs and a consumer
> queries it, the container sits idle by design. Skip this phase until a consumer
> is on the horizon if you would rather not spend the RAM now.

## Prerequisites

- Access to the Proxmox host shell — **pve → Shell** in the web UI, or
  `ssh root@192.168.2.210` (see the [NUC](../hardware/nuc.md))
- The static IP reserved for the Dragonfly-LXC: `192.168.2.214` (the
  [network overview](../network/overview.md) is the source of truth for
  addresses; reserve it per [Reserve an IP Address in the KPN Box](reserve-ip-kpn-box.md))

All `pct` commands run on the Proxmox host, not inside the container.

## 1. Create the LXC

Use an **unprivileged** container on **Debian 13** with the rootfs on the NVMe.
No UID/GID map is needed — Dragonfly writes only to its local rootfs, never the
NAS.

1. Fetch the template if it is not already present (**pve → local → CT
   Templates**, or on the host):

   ```bash
   pveam update
   pveam download local debian-13-standard_13.1-2_amd64.tar.zst
   ```

2. Create the container via **Create CT**, with these non-default settings:

   - **General**: CT ID `214` (matches the IP octet), Hostname `dragonfly`,
     **Unprivileged container** enabled.
   - **Template**: storage `local`, the `debian-13-standard` image.
   - **Disks**: rootfs on `local-lvm` (the NVMe), `8` GiB — the binary plus room
     for snapshot files; the working set lives in RAM.
   - **CPU**: `2` cores.
   - **Memory**: `4096` MiB RAM, `512` MiB swap. The cache is capped below this
     (step 3) so the process cannot push the host into swap.
   - **Network**: bridge `vmbr0`, IPv4 **Static** `192.168.2.214/24`, gateway
     `192.168.2.254`.

   Do not start it yet.

3. Enable the `nesting` feature (Debian 13 / systemd 257, as on the other
   guests):

   ```bash
   pct set 214 -features nesting=1
   ```

### Verify

```bash
pct start 214
pct exec 214 -- systemctl is-system-running    # running (or degraded — check systemctl --failed)
pct exec 214 -- ip -4 addr show eth0            # 192.168.2.214/24
pct exec 214 -- ping -c1 192.168.2.254          # gateway
```

## 2. Install Dragonfly

Dragonfly ships a single static binary — the same static-binary-plus-systemd
approach the corpus platform uses. Install it from the GitHub release, pinning a
version rather than tracking `latest`.

```bash
pct enter 214
apt-get update && apt-get install -y curl redis-tools    # redis-tools gives redis-cli for the smoke test

ver=v1.24.0                                               # pin the release; check github.com/dragonflydb/dragonfly/releases
curl -fsSL -o /tmp/dragonfly.tar.gz \
  "https://github.com/dragonflydb/dragonfly/releases/download/${ver}/dragonfly-x86_64.tar.gz"
tar -xzf /tmp/dragonfly.tar.gz -C /tmp
install -m 0755 /tmp/dragonfly-x86_64 /usr/local/bin/dragonfly
dragonfly --version

useradd -r -s /usr/sbin/nologin dragonfly
install -d -o dragonfly -g dragonfly /var/lib/dragonfly
```

## 3. Run Dragonfly as a systemd unit

Bind to the container's LAN address so other guests can reach it, require a
password, and cap the cache at 3 GB (below the 4 GB guest RAM).

```bash
cat > /etc/systemd/system/dragonfly.service <<'EOF'
[Unit]
Description=Dragonfly (Feast online store)
After=network-online.target
Wants=network-online.target

[Service]
User=dragonfly
Group=dragonfly
ExecStart=/usr/local/bin/dragonfly \
  --bind 192.168.2.214 --port 6379 \
  --requirepass CHANGE_ME \
  --maxmemory 3gb \
  --proactor_threads 2 \
  --dir /var/lib/dragonfly --dbfilename dump
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now dragonfly
```

> [!NOTE]
> Dragonfly prefers `io_uring`, which an unprivileged LXC's seccomp filter may
> block — the service then fails to start with an `io_uring` error. If so, add
> `--force_epoll` to `ExecStart` and `systemctl restart dragonfly`; the epoll
> backend works everywhere at a small throughput cost, irrelevant at this scale.

### Verify

Locally on the container, then across the LAN from another guest — the real path
a consumer takes:

```bash
# on CT 214
redis-cli -h 192.168.2.214 -a CHANGE_ME PING            # PONG
redis-cli -h 192.168.2.214 -a CHANGE_ME SET k v         # OK
redis-cli -h 192.168.2.214 -a CHANGE_ME GET k           # "v"
```

```bash
# from the MLflow LXC (CT 213) — proves LAN reachability
pct exec 213 -- bash -c 'apt-get install -y redis-tools >/dev/null; redis-cli -h 192.168.2.214 -a CHANGE_ME PING'   # PONG
```

Feast reaches it with a Redis online-store URL —
`redis://:CHANGE_ME@192.168.2.214:6379/0` — configured in predict's
`feature_store.yaml`. That wiring is a predict-repo concern, not part of this
host setup.

## 4. Backups

A `vzdump` of this LXC captures the binary and unit, but **not** a meaningful
data recovery point — the online store is a cache. Its true recovery is a re-run
of `feast materialize` from the offline parquet, which is authoritative. Back up
the container for config convenience, not for the data.

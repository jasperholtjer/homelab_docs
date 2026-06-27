# Deploy the DB-VM (Postgres + Neo4j + serving tier)

Stand up the consumer of the [eve-industry-corpus](../projects/eve-industry-corpus.md)
platform — phase 5 of that project's build order. The DB-VM holds the Postgres
`eve` database and Neo4j, and runs the `eve-industry-serving` loader that reads
`_DONE` Gold partitions into both stores. The Dagster LXC triggers the loader
over SSH.

The serving application code (loader, schema DDL, deploy scripts) lives in the
`eve-industry-serving` repository and is pulled in during the runtime step — this
guide covers the homelab-side host setup, not the application internals.

## Prerequisites

- Access to the Proxmox host shell — `ssh root@192.168.2.210` (see the [NUC](../hardware/nuc.md))
- The static IP reserved for the DB-VM: `192.168.2.212` (the
  [network overview](../network/overview.md) is the source of truth for addresses)
- Phases 1–4 done: the UNAS exports the `EVE` share over NFS and Gold partitions
  exist (see [Deploy the Dagster Orchestration LXC](deploy-dagster-lxc.md))
- A read-only deploy key on the `eve-industry-serving` GitHub repo (created below)

The DB-VM is a full **VM**, so guest steps run over SSH or the Proxmox console;
`qm` commands run on the host.

## 1. Create the VM

Create it via **Create VM** (Proxmox 9) with these non-default settings:

- **General**: VM ID `212`, Name `db-vm`.
- **OS**: the Debian 13 netinst ISO (download via `local → ISO Images → Download
  from URL`), type Linux.
- **System**: SCSI controller `VirtIO SCSI single`, **Qemu Agent** enabled.
- **Disks**: `120` GiB on `local-lvm` (the NVMe), **Discard** enabled. The DB
  files never live on the UNAS (single HDD over NFS has poor random IO and risky
  fsync semantics).
- **CPU**: `2` cores, type **`host`** (single-node, no migration → best DB perf).
- **Memory**: `12288` MiB (Postgres + Neo4j heap 2 GB + page cache; leaves headroom
  for the Dagster LXC's ~12 GiB backfill peak on the 32 GB NUC).
- **Network**: bridge `vmbr0`, model VirtIO.

Install Debian over the console (noVNC). Choices that matter:

- **Network**: hostname `db-vm`. The installer's manual static config may not stick
  (it falls back to DHCP); that is fixed in step 2.
- **Users**: set a root password and create an admin user (e.g. `jasper`).
- **Partitioning**: guided, entire disk, all files in one partition.
- **Software selection**: deselect every desktop; keep only **SSH server** and
  **standard system utilities**. (This minimal set omits `sudo`, `gpg`, `curl`,
  `git` — installed in step 2.)
- **GRUB**: install to `/dev/sda`.

Before the post-install reboot, detach the ISO: **VM 212 → Hardware → CD/DVD Drive
→ Edit → Do not use any media**, so it boots from disk.

## 2. Base setup and static IP

Log in over the console as `root` (the fresh admin user is not yet in `sudo`).
Connect a workstation editor later via SSH config:

```sshconfig
Host eve-db-vm
  HostName 192.168.2.212
  User jasper
```

As root on the VM, install the base packages and grant the admin user `sudo`
(group change applies on next login):

```bash
apt-get update && apt-get install -y sudo gpg curl git nfs-common qemu-guest-agent
usermod -aG sudo jasper
```

Set the static IP (the interface is `ens18`):

```bash
cp /etc/network/interfaces /root/interfaces.bak
cat > /etc/network/interfaces <<'EOF'
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
    address 192.168.2.212/24
    gateway 192.168.2.254
EOF
```

`dhcpcd` regenerates `/etc/resolv.conf` on boot and wipes a hand-written
`nameserver`, which breaks DNS. Pin resolvers via `resolv.conf.head` (always
prepended), then reboot:

```bash
printf 'nameserver 1.1.1.1\nnameserver 8.8.8.8\n' > /etc/resolv.conf.head
reboot
```

### Verify

```bash
ssh eve-db-vm 'ip -4 addr show ens18; ping -c1 8.8.8.8; getent hosts deb.debian.org'
```

`192.168.2.212/24` on `ens18`, ping OK, and name resolution OK.

## 3. Mount the Gold tree over NFS (read-only)

The serving tier only ever reads Gold. First **broaden the UNAS NFS export** to
permit the VM: in the UniFi UNAS UI (`http://192.168.2.24`), open the `EVE` share's
NFS settings and add client `192.168.2.212` (the export is otherwise scoped to the
host `192.168.2.210` only). Keep the export **sync** — `_DONE` must mean the data
is durably on disk for the producer.

Find the export path and mount it read-only (`nfs-common` is already installed):

```bash
showmount -e 192.168.2.24    # confirms .212 is now permitted; copy the export path
mkdir -p /mnt/eve
cp /etc/fstab /etc/fstab.bak
echo '192.168.2.24:/volume/<drive-uuid>/.srv/.unifi-drive/EVE/.data  /mnt/eve  nfs  ro,defaults,_netdev,vers=3  0  0' >> /etc/fstab
systemctl daemon-reload && mount /mnt/eve
```

Create the `serving` service account in group **988** (the UNAS data group) so it
can read the share, and confirm the read path:

```bash
getent group 988 || groupadd -g 988 nfsdata
useradd -m -g 988 -s /bin/bash serving       # uid is auto-assigned; only group 988 matters for NFS
su - serving -c 'cat /mnt/eve/gold/sde-categories/_INDEX.json | head -c 80; echo " READ-OK"'
```

> [!NOTE]
> 988 is the UniFi OS data group today; confirm against `ls -n /mnt/eve` and
> substitute if the firmware differs. Files read back as owner `nobody` — cosmetic;
> read access hinges on group 988.

## 4. Install Postgres

Postgres listens on `localhost` only (the loader is co-located). Create the `eve`
database and the `serving` role (pick a password; it goes in `serving.env`):

```bash
apt-get install -y postgresql
sudo -u postgres psql -c "CREATE ROLE serving LOGIN PASSWORD 'CHANGE_ME';"
sudo -u postgres createdb -O serving eve
sudo -u postgres psql -d eve -c '\conninfo'
```

## 5. Install Neo4j

Add the Neo4j apt repo, install, cap the heap, and bind to localhost so the JVM
cannot push the host into swap:

```bash
install -d /etc/apt/keyrings
wget -qO- https://debian.neo4j.com/neotechnology.gpg.key | gpg --dearmor -o /etc/apt/keyrings/neo4j.gpg
echo 'deb [signed-by=/etc/apt/keyrings/neo4j.gpg] https://debian.neo4j.com stable latest' > /etc/apt/sources.list.d/neo4j.list
apt-get update && apt-get install -y neo4j

conf=/etc/neo4j/neo4j.conf
sed -i 's/^#\?server.memory.heap.initial_size=.*/server.memory.heap.initial_size=2g/' $conf
sed -i 's/^#\?server.memory.heap.max_size=.*/server.memory.heap.max_size=2g/' $conf
sed -i 's/^#\?server.default_listen_address=.*/server.default_listen_address=localhost/' $conf

neo4j-admin dbms set-initial-password 'CHANGE_ME'    # must run before first start
systemctl enable --now neo4j
```

### Verify

```bash
cypher-shell -a bolt://localhost:7687 -u neo4j -p 'CHANGE_ME' 'RETURN 1'
```

## 6. Deploy the serving tier

The repo is private, so authorize the VM with a **read-only deploy key**.

1. As `serving`, generate a key and print the public half:

   ```bash
   su - serving -c 'ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -q; cat ~/.ssh/id_ed25519.pub'
   ```

   On GitHub: repo → **Settings → Deploy keys → Add deploy key**, paste it, leave
   **Allow write access unchecked**.

2. Install `uv` for `serving`, then clone the repo (owned by `serving`) and check
   out `develop`:

   ```bash
   su - serving -c 'curl -LsSf https://astral.sh/uv/install.sh | sh'
   install -d -o serving -g nfsdata /opt/eve-industry-serving
   su - serving -c 'git clone git@github.com:jasperholtjer/eve-industry-serving.git /opt/eve-industry-serving && cd /opt/eve-industry-serving && git checkout develop'
   ```

3. Write the runtime config (root-owned, group-readable by `serving`) and fill in
   the Postgres + Neo4j passwords:

   ```bash
   install -d -m 0750 -o root -g nfsdata /etc/eve-serving
   install -m 0640 -o root -g nfsdata \
     /opt/eve-industry-serving/deploy/serving.env.example /etc/eve-serving/serving.env
   nano /etc/eve-serving/serving.env
   ```

4. Run the redeploy and put it on PATH:

   ```bash
   bash /opt/eve-industry-serving/deploy/redeploy.sh
   ln -s /opt/eve-industry-serving/deploy/redeploy.sh /usr/local/bin/redeploy
   ```

   `redeploy` pulls the latest code as `serving`, runs `uv sync --frozen --no-dev
   --extra loader` (the loader's `duckdb`/`psycopg`/`neo4j` deps), applies the DDL
   via `eve-serving db init` (idempotent), and installs the
   `/usr/local/bin/eve-serving` PATH wrapper.

> [!NOTE]
> `redeploy` reads itself into memory before its `git pull`. When a pull changes
> `redeploy.sh` itself, that run still uses the old version — run `redeploy` a
> second time to apply the new script.

### Verify

Load the SDE (market FKs depend on it), then a market dataset, as `serving`:

```bash
su - serving -c 'eve-serving load --dataset sde --latest'
su - serving -c 'eve-serving load --dataset market-history'
su - serving -c 'eve-serving load --dataset market-orders-live'
su - serving -c 'eve-serving load --dataset market-prices-live'
sudo -u postgres psql -d eve -c 'SELECT count(*) FROM market.orders_live;'   # > 0
```

A second run of any dataset reports `skipped` (idempotent on `parquet_sha256`).

## 7. Wire the SSH trigger from the Dagster LXC

The orchestrator (CT 211) invokes the loader as `serving` on the DB-VM, authorized
by the LXC's `corpus` account.

1. Print the `corpus` public key on the LXC (`pct enter 211`):

   ```bash
   su - corpus -c 'test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -q; cat ~/.ssh/id_ed25519.pub'
   ```

2. Authorize it for `serving` on the DB-VM (root):

   ```bash
   install -d -m 700 -o serving -g nfsdata /home/serving/.ssh
   echo '<corpus-pubkey>' >> /home/serving/.ssh/authorized_keys
   chmod 600 /home/serving/.ssh/authorized_keys && chown -R serving:nfsdata /home/serving/.ssh
   ```

### Verify

From the LXC, **as `corpus`** (not root — only the corpus key is authorized):

```bash
su - corpus
ssh -o StrictHostKeyChecking=accept-new serving@192.168.2.212 eve-serving load --dataset market-prices-live
```

A `skipped` (or `loaded`) result confirms the full chain: SSH trust, the PATH
wrapper, the env file, and the DB connection.

## 8. Backups

`vzdump` the VM to the UNAS backup storage on the same schedule as the LXC. The
databases live on the NVMe; the dump is the recovery point.

# DB-VM — Postgres + Neo4j Serving Host

Proxmox guest that hosts the databases consumed by the
[eve-industry-corpus](../projects/eve-industry-corpus.md) platform: the Postgres
`eve` database and Neo4j (star-map). It also runs the `eve-industry-serving`
loader, which reads `_DONE` Gold partitions into both stores. This is a **virtual
guest**, not physical hardware — it runs on the [NUC](nuc.md) NVMe.

## Access

- IP: `192.168.2.212` (static, on bridge `vmbr0`)
- SSH: `ssh jasper@192.168.2.212` (admin user; has `sudo`)
- VM ID: `212`, hostname `db-vm`
- Console: Proxmox host → `VM 212 → Console` (noVNC), or `ssh root@192.168.2.210`
  then `qm terminal 212`

Both databases bind to `localhost` only — they are **not** reachable on
`192.168.2.212`. The serving loader is co-located and SSH-triggered from the
Dagster LXC (CT 211). To query a database from a workstation, tunnel over SSH:

```bash
ssh -L 5432:localhost:5432 -L 7687:localhost:7687 jasper@192.168.2.212
```

## Services

| Service  | Bind           | Details                                                     |
| -------- | -------------- | ---------------------------------------------------------- |
| Postgres | localhost:5432 | Database `eve`, login role `serving`                       |
| Neo4j    | localhost:7687 | Bolt; user `neo4j`, heap capped at 2 GB                    |
| Loader   | —              | `eve-serving` CLI, run as `serving`; SSH-triggered by CT 211 |

The `serving` OS account (group 988, the UNAS data group) owns the loader and the
read-only Gold mount at `/mnt/eve`.

## Virtual hardware

| Component | Spec                                                          |
| --------- | ------------------------------------------------------------- |
| VM ID     | 212                                                          |
| CPU       | 2 cores, type `host`                                         |
| Memory    | 12288 MiB (Postgres + Neo4j 2 GB heap + page cache)          |
| Disk      | 120 GiB on `local-lvm` (NUC NVMe), discard enabled          |
| Network   | bridge `vmbr0`, VirtIO, static `192.168.2.212/24`           |
| Storage   | Gold tree NFS-mounted read-only at `/mnt/eve` from the UNAS |

Database files live on the NVMe only — never on the UNAS (single HDD over NFS has
poor random IO and risky fsync semantics). The NFS mount is read-only Gold input.

## Software

- Debian 13 (minimal: SSH server + standard utilities)
- Postgres, Neo4j
- `uv` + the `eve-industry-serving` loader (checked out under `/opt`)

## Backups

`vzdump` the VM to the UNAS backup storage on the same schedule as the LXCs; the
dump is the recovery point (DB files are on the ephemeral NVMe).

## Reference

- [Deploy the DB-VM](../howto/deploy-db-vm.md) — full build runbook (phase 5)
- [eve-industry-corpus deployment](../projects/eve-industry-corpus.md) — role split and topology
- [Network overview](../network/overview.md) — IP plan
- [NUC](nuc.md) — the Proxmox host it runs on

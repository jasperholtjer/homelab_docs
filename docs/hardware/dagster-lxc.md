# Dagster-LXC — Orchestration Host

Proxmox guest that runs the Dagster orchestrator and the static `corpus` binary
for the [eve-industry-corpus](../projects/eve-industry-corpus.md) platform — the
producer's control plane. It mounts the UNAS `EVE` share over NFS, drives the
`corpus` subcommands (ingest → Silver → Gold), and keeps Dagster's run-state on
the NUC NVMe. This is a **virtual guest**, not physical hardware — it runs on the
[NUC](nuc.md). The orchestration code lives in the `eve-industry-orchestration`
repository.

## Access

- IP: `192.168.2.211` (static, on bridge `vmbr0`)
- Web UI: [192.168.2.211:3000](http://192.168.2.211:3000) (Dagster)
- CT ID: `211`, hostname `dagster`, unprivileged
- Shell: `ssh root@192.168.2.210` then `pct enter 211` (guests have no direct SSH)

Container root cannot write `/mnt/eve` (it is not in group 988). Run all
share-touching work as the `corpus` account.

## Services

| Service   | Bind                 | Details                                                  |
| --------- | -------------------- | -------------------------------------------------------- |
| Webserver | 192.168.2.211:3000   | `dagster-webserver.service`, runs as `corpus`            |
| Daemon    | —                    | `dagster-daemon.service`; sensors, schedules, run-queue  |
| corpus    | —                    | static binary at `/usr/local/bin/corpus`; does the compute |

`QueuedRunCoordinator` caps `max_concurrent_runs: 4` (the NAS-spindle I/O cap),
with `heavy` / `everef_download` concurrency pools bounding Gold memory and EVE
Ref fetches.

## Environment

The systemd units carry these via `Environment=` (mirrored in the repo `.env` for
local `dg dev`). No credentials — the serving-load assets SSH to the DB-VM using
the `corpus` account's authorized key.

| Variable              | Value                              | Purpose                                  |
| --------------------- | ---------------------------------- | ---------------------------------------- |
| `DAGSTER_HOME`        | `/var/lib/dagster`                 | Run-state (SQLite) on the NVMe           |
| `CORPUS_BINARY_PATH`  | `/usr/local/bin/corpus`            | Static compute binary                    |
| `CORPUS_DATASETS_DIR` | `/usr/local/share/corpus/datasets` | Dataset YAML configs                     |
| `CORPUS_SINK_PATH`    | `/mnt/eve`                         | NFS sink for the `_DONE` contract        |
| `RAYON_NUM_THREADS`   | `6`                                | Caps the market-orders parser below core count |
| `SERVING_HOST`        | `192.168.2.212` (default)          | DB-VM the `eve-serving load` CLI runs on |
| `SERVING_USER`        | `serving` (default)                | SSH account with the loader on PATH      |

## Virtual hardware

| Component | Spec                                                          |
| --------- | ------------------------------------------------------------- |
| CT ID     | 211 (unprivileged, `nesting=1`)                             |
| CPU       | 2 cores                                                      |
| Memory    | 4096 MiB RAM + 512 MiB swap                                  |
| Disk      | 20 GiB rootfs on `local-lvm` (NUC NVMe)                     |
| Network   | bridge `vmbr0`, static `192.168.2.211/24`                  |
| Storage   | UNAS `EVE` share NFS-mounted at `/mnt/eve` (host mount, bind-mounted in) |

Run-state (SQLite `DAGSTER_HOME`) lives on the NVMe, decoupled from the DB-VM
(no boot-order dependency). The share holds only the parquet contract, not DB
files. The `corpus` service account is in group **988** (the UNAS data group) so
its writes to the share are readable by the UNAS.

## Software

- Debian 13 (unprivileged LXC template)
- `uv` + the `eve-industry-orchestration` venv (under `/opt`)
- The static `corpus` binary + dataset bundle (installed by `redeploy.sh`)

## Backups

`vzdump` the LXC to the UNAS backup storage on the same schedule as the DB-VM; the
dump is the recovery point (run-state is on the ephemeral NVMe).

## Reference

- [Deploy the Dagster Orchestration LXC](../howto/deploy-dagster-lxc.md) — build runbook (phase 3)
- [Install the corpus Binary on the Dagster LXC](../howto/install-corpus-binary.md)
- [eve-industry-corpus deployment](../projects/eve-industry-corpus.md) — role split and topology
- [Network overview](../network/overview.md) — IP plan
- [DB-VM](db-vm.md) — the serving-tier host it triggers loads on
- [NUC](nuc.md) — the Proxmox host it runs on

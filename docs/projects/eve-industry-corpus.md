# eve-industry-corpus — Homelab Deployment

How the `eve-industry-corpus` data platform runs on this homelab: which guests
host it, how they are wired to storage, and the decisions and build order behind
the setup.

Scope is the **homelab side only**. The platform design — medallion model,
partition contract, schema policy, and the ADRs that govern them — lives in the
separate `eve-industry-corpus` repository and is the single source of truth for
those. This page links to the hardware it runs on rather than restating it.

## Role split

The Rust `corpus` binary does the work: ingest → Silver → Gold, with SQLite
run-state and the `parquet + _INDEX.json + _DONE` contract written to the NAS.
Dagster is a **thin orchestrator** — it owns the partition matrix, the backfill
UI, and the materialisation log; it shells out to the `corpus` subcommands.

- Raw and materialised data (Bronze/Silver/Gold parquet) live on the [UNAS 2](../hardware/unas.md).
- Databases (the downstream consumer, and Dagster's own state) live on the [NUC](../hardware/nuc.md) NVMe.
- The Dagster guest is the only one that touches both the UNAS (NFS) and the consumer database.

## Topology

```text
┌─ NUC / Proxmox (NVMe) ─────────────────────────────────────┐
│                                                             │
│  [DB-VM]   Postgres (db `eve`) + Neo4j   ← downstream       │
│              ▲  consumer of Gold parquet                    │
│              │ vmbr0                                         │
│  [Dagster-LXC]  webserver + daemon + corpus binary          │
│              │  SQLite run-state + Dagster storage (local)  │
│              │  NFS bind-mount ──────────┐                  │
└──────────────┼──────────────────────────┼──────────────────┘
               │ DB / queries             │ NFS (2.5G switch)
               ▼                          ▼
          DB-VM                     [UNAS 2]  parquet + _DONE + vzdump
```

## Decisions

| Decision | Choice | Why |
| --- | --- | --- |
| Orchestrator | Dagster, thin | Backfill UI and partition dependencies for a multi-year, multi-dataset backfill; the binary keeps the compute and run-state. |
| Dagster storage | Local SQLite in the LXC | Decouples the orchestrator from the DB-VM (no boot-order dependency); load is light enough that SQLite locking is a non-issue. |
| Execution | Run the static `corpus` binary directly | A static binary has no runtime dependencies; an LXC and Dagster's run-queue already provide the isolation and limits a container would add. |
| Distribution | Pinned binary, pulled once per version | The artefact is built once per release and cached locally; runs execute from the local copy with no per-run network dependency. The build/publish mechanism is a CI decision in the `eve-industry-corpus` repo. |
| Storage protocol | NFS | NFS outperforms SMB for this workload and is the Proxmox-native choice. |
| Databases | Postgres + Neo4j in one VM | Strong isolation and clean full-snapshot backups; raw data stays off the database entirely. |

> [!WARNING]
> Live database files never live on the UNAS. A single HDD over NFS gives poor
> random IO and risky lock/fsync semantics. The UNAS holds the raw and parquet
> data; the databases read from the NVMe. Only raw input and backups go to the UNAS.

## Build order

Producer-first: each phase is independently verifiable, and the consumer
database comes only once the data product exists.

1. **UNAS storage.** Create a dedicated Shared Drive (`corpus`), enable the NFS
   export scoped to the LAN subnet. Expect the UniFi UID/GID quirk (`unifi-core`,
   988). _Verify:_ a Linux client can mount and write.
2. **Proxmox host.** Add the UNAS NFS as Proxmox storage (`backup`, optionally
   `iso`/`vztmpl`). _Verify:_ the storage shows online; a test write succeeds.
3. **Dagster-LXC + corpus.** Create the LXC (rootfs on NVMe). Host-mount the NFS
   and bind-mount it into the container; fix permissions for the 988 quirk.
   Install the pinned `corpus` binary, `uv`/Python + Dagster, with `webserver`
   and `daemon` as systemd units and `DAGSTER_HOME` on SQLite storage. Wire the
   Dagster assets to the `corpus` subcommands, partitioned per dataset.
   _Verify:_ one partition runs end-to-end and writes `parquet + _INDEX.json + _DONE`.
4. **Backfill orchestration.** Set the run-queue concurrency deliberately low —
   polite to the upstream source and gentle on the single-HDD UNAS. Each dataset
   declares its own partition scheme and any cross-partition dependencies (these
   vary per dataset; the platform setup is identical regardless). _Verify:_ a
   small backfill produces a correct contract and is idempotent on re-run.
5. **DB-VM (consumer).** Create the VM (disk on NVMe). Install Postgres (listening
   on `vmbr0`, `pg_hba` opened for the Dagster-LXC subnet, database `eve`) and
   Neo4j (with an explicit heap limit). Add a loader that reads `_DONE`
   partitions into the databases. _Verify:_ a loaded Gold partition is queryable.
6. **Backups and ops.** `vzdump` the VM and LXC to the UNAS storage. Document the
   runbook: recover a stuck run, replay a partition, resume a backfill.

Critical path: phases 1 → 2 → 3 → 4 yield a working producer. Phases 5 and 6
hang off it and can follow later.

## Capacity

The NUC's 32 GB RAM is the binding constraint. Neo4j is the swing factor — give
it an explicit heap limit (`server.memory.heap.max_size`) so the JVM does not
push the host into swap. Rough budget:

| Consumer | Estimate |
| --- | --- |
| Proxmox host | ~2 GB |
| DB-VM (Postgres 2–4 GB + Neo4j heap 2 GB + page cache 2 GB) | ~8 GB |
| Dagster-LXC (baseline + concurrent runs) | 2–4 GB |
| Headroom / peak | remainder |

## Open items

- IP addresses for the Dagster-LXC and DB-VM are not yet assigned. Record them in
  the [network overview](../network/overview.md) once chosen; it stays the single
  source of truth for addresses.
- The UNAS share name (`corpus` assumed above) is not yet created.
- The binary build/publish mechanism (GitHub Release asset vs GHCR via `oras`) is
  decided in the `eve-industry-corpus` repository, not here.

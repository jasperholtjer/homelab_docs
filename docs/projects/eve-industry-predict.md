# eve-industry-predict — Homelab Deployment

How the `eve-industry-predict` ML platform runs on this homelab: which guests
host it, how they are wired to storage, and the decisions and build order behind
the setup.

Scope is the **homelab side only**. The training design — questions, models,
Feast feature recipe, promotion policy — lives in the separate
`eve-industry-predict` repository and is the single source of truth for those.
This page links to the hardware it runs on rather than restating it.

## Role split

`predict` is the **training tier**. It is a read-only consumer of the corpus Gold
contract and ships two shared outputs: **MLflow pyfunc artefacts** (model +
inference wrapper) and **Feast feature definitions**. It runs **no long-lived
HTTP service of its own** — training happens on the workstation GPU; the homelab
only hosts the always-on platform services training and inference lean on.

Two of those services need a host on the NUC; the rest are files on the NAS:

- **MLflow tracking + registry** — always-on server with its own Postgres
  backing store. The workstation logs runs to it; consumers resolve model
  versions from it.
- **Feast online store (Dragonfly)** — an in-memory feature cache read by
  inference-time consumers. Provisioned ahead of demand (see _Open items_).
- **MLflow artefact root** and **Feast registry + offline parquet** are
  **files**, not processes — they live on a dedicated UNAS Shared Drive (`ML`),
  separate from the corpus medallion.

> [!IMPORTANT]
> Predict must never write into the `EVE` share — that is the source-faithful
> corpus medallion it consumes read-only. Model-derived artefacts (MLflow
> models, Feast parquet) land on their own `ML` share. Keeping the two apart is
> the storage-level expression of the consumer boundary.

## Topology

```text
┌─ NUC / Proxmox (NVMe) ─────────────────────────────────────┐
│                                                             │
│  [MLflow-LXC 213]  MLflow server + Postgres (metadata)      │
│        :5000          │  --serve-artifacts (HTTP proxy)     │
│        │              │  NFS bind-mount → ML share          │
│        │              └──────────────────────┐             │
│  [Dragonfly-LXC 214]  Feast online store       │            │
│        :6379          in-memory, no NFS         │           │
└────────┼──────────────────────────────────────┼────────────┘
         │ HTTP (tracking + artefacts)           │ NFS (2.5G switch)
         ▼                                       ▼
   Workstation (WiFi)                       [UNAS 2]  ML share:
   MLFLOW_TRACKING_URI                        mlflow-artifacts/
   = http://192.168.2.213:5000                feast/ (registry + offline parquet)
```

The workstation reaches MLflow over plain HTTP on the shared `/24`. Artefacts are
**proxied through the MLflow server** (`--serve-artifacts`), so no client needs
NFS or SMB to push a model — only the server mounts the NAS. The Feast offline
parquet and registry on the same `ML` share are read/written by the workstation
directly over SMB, the way it already reaches corpus Gold.

## Decisions

| Decision | Choice | Why |
| --- | --- | --- |
| Guest type | Two unprivileged **LXCs**, not VMs | Light always-on services; matches the Dagster-LXC pattern. A VM's snapshot isolation is only worth it for the databases. |
| Split | MLflow and Dragonfly in **separate** containers | Different planes (training vs serving), lifecycles, and RAM profiles. A Dragonfly restart is a cold cache; MLflow's is not. LXC overhead is cheap enough that splitting costs almost nothing. |
| MLflow backing store | **Own Postgres inside the LXC** | Self-contained, no boot-order dependency on the DB-VM. Matches predict charter §7.1. |
| Artefact access | **Proxied** via `mlflow server --serve-artifacts` | Clients need only HTTP; the NAS is mounted once, on the server. No NFS/SMB on Windows for artefact push. |
| Artefact + Feast storage | Dedicated UNAS Shared Drive **`ML`** | Keeps model-derived files off the `EVE` medallion share, honouring predict's read-only-consumer boundary. Separate quota and backup. |
| Feast registry / offline store | **Parquet files** on the `ML` share | No process. `feast apply` from the workstation updates the registry; `FileSource` reads the offline parquet. |

## Build order

Platform-first: stand up the always-on tracking server, then the online store
that inference will later read.

1. **UNAS `ML` share.** Create a dedicated Shared Drive (`ML`), enable NFS scoped
   to the MLflow LXC host, and grant the workstation SMB access for the Feast
   parquet. Expect the UniFi UID/GID quirk (`unifi-core`, 988). _Verify:_ the
   Proxmox host can mount and write; the workstation can browse the SMB share.
2. **MLflow-LXC (CT 213).** Create the LXC (rootfs on NVMe), NFS-mount the `ML`
   share and bind it in, install Postgres (database `mlflow`) and MLflow as a
   systemd unit with `--serve-artifacts`. See
   [Deploy the MLflow LXC](../howto/deploy-mlflow-lxc.md). _Verify:_ the
   workstation logs a run and an artefact end-to-end.
3. **Dragonfly-LXC (CT 214).** Create the LXC, install Dragonfly as a systemd
   unit bound to the LAN with a password and a memory cap. See
   [Deploy the Dragonfly LXC](../howto/deploy-dragonfly-lxc.md). _Verify:_ a
   `redis-cli PING` from another guest returns `PONG`.
4. **Backups.** `vzdump` both LXCs to the UNAS backup storage. Postgres metadata
   lives on the NVMe; the dump is its recovery point. Dragonfly is a cache — its
   recovery point is a re-run of `feast materialize`, not a backup.

Critical path: phases 1 → 2 yield a working tracking + registry server, which is
all training needs. Phase 3 hangs off it and can follow when a consumer is ready.

## Capacity

The NUC's 32 GB RAM is the binding constraint. Rough allocation after this
project lands, alongside the corpus guests:

| Consumer | Estimate |
| --- | --- |
| Proxmox host | ~2 GB |
| DB-VM (212) | ~12 GB allocated |
| Dagster-LXC (211) | ~4 GB |
| MLflow-LXC (213) — MLflow ~300 MB + Postgres ~500 MB + OS | ~2 GB |
| Dragonfly-LXC (214) — cache cap 3 GB + OS | ~4 GB |
| Headroom / peak | remainder (~8 GB) |

Dragonfly is the swing factor — cap it (`--maxmemory 3gb`) so the cache cannot
push the host into swap, exactly as Neo4j's heap is capped on the DB-VM.

## Open items

- **When does the online store earn its keep?** Nothing reads Dragonfly today —
  no consumer serves model output yet (`eve-industry-serving` serves corpus
  facts, not forecasts). It is provisioned ahead of the predict charter's "after
  Q5" trigger (open Q 7.6). Until `feast materialize` runs and a consumer queries
  it, CT 214 sits idle by design.
- **Where forecasts land** is an open cross-repo design item (predict charter
  §7, open Q 0.1) and does not block this platform: predict ships MLflow
  artefacts and Feast definitions, not forecast tables.

# Homelab Docs

Living reference for the homelab: hardware inventory, network layout, and how-to
guides. Kept small and practical so infrequently-used knowledge stays close at hand.

## Overview

| Device      | Role                  | Address                                              | Notes                          |
| ----------- | --------------------- | ---------------------------------------------------- | ------------------------------ |
| KPN Box 12  | Router / gateway      | 192.168.2.254                                        | Internet + LAN gateway         |
| Workstation | Daily-driver PC       | —                                                    | Windows 11 Home N, WiFi        |
| NUC         | Virtualization host   | [192.168.2.210:8006](https://192.168.2.210:8006)     | Proxmox VE 9, switch port 4    |
| UNAS 2      | Network storage (NAS) | [192.168.2.24](http://192.168.2.24/)                 | 24 TB single disk, switch port 1 |
| Flex Mini   | 2.5G switch           | —                                                    | Uplinks NUC and UNAS 2         |

## Documentation

- [Network overview](docs/network/overview.md) — IP plan and topology
- Hardware
  - [NUC](docs/hardware/nuc.md) — Proxmox host
  - [Dagster-LXC](docs/hardware/dagster-lxc.md) — orchestration guest (eve-industry-corpus)
  - [DB-VM](docs/hardware/db-vm.md) — Postgres + Neo4j guest (eve-industry-corpus)
  - [UNAS 2](docs/hardware/unas.md) — network storage
  - [Flex Mini switch](docs/hardware/switch.md) — 2.5G switching
  - [Workstation](docs/hardware/workstation.md) — daily-driver PC
- Projects
  - [eve-industry-corpus](docs/projects/eve-industry-corpus.md) — data platform deployment
  - [eve-industry-predict](docs/projects/eve-industry-predict.md) — ML platform deployment (MLflow + Feast online store)
- [How-to guides](docs/howto/) — task-oriented runbooks

## Conventions

- One file per device under `docs/hardware/`.
- How-to guides are task-oriented and live under `docs/howto/`.
- Record every change in [CHANGELOG.md](CHANGELOG.md).

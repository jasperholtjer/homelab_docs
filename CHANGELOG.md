# Changelog

All notable changes to this documentation are recorded here. Format based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Initial repository structure: README index, network overview, hardware docs, how-to section
- Hardware inventory for NUC (Proxmox VE 9), UNAS 2, Flex Mini 2.5G switch, and workstation
- Network overview with IP plan and topology diagram
- CLAUDE.md with repository usage rules and content-adding workflow
- KPN Box 12 gateway, switch port assignments (UNAS port 1, NUC port 4), and workstation WiFi connection
- How-to guide: update Proxmox VE via the web UI
- How-to guide: reserve an IP address in the KPN Box via DHCP reservation
- Project deployment doc for eve-industry-corpus (Dagster-LXC, DB-VM, UNAS NFS, build order)
- `docs/projects/` section with layout and content-adding rules in CLAUDE.md
- Static IPs for Dagster-LXC (192.168.2.211) and DB-VM (192.168.2.212) in the network IP plan
- How-to guide: add a UNAS NFS export as Proxmox storage
- How-to guide: deploy the Dagster orchestration LXC — unprivileged container, 988 UID/GID map, UNAS NFS host-mount and bind-mount (eve-industry-corpus phase 3)
- How-to guide: install the corpus binary on the Dagster LXC — fine-grained PAT auth, pinned checksum-verified release, CORPUS_DATASETS_DIR, end-to-end ingest smoke test (v0.1.3)
- Deploy-Dagster-LXC how-to: section 4 "Deploy the orchestrator" — user-local `uv` install as `corpus`, clone + `uv sync`, `DAGSTER_HOME` setup, systemd unit install, and a real-binary Silver smoke test
- NUC Access section: SSH entry point and the `ssh root@210` → `pct enter 211` two-hop to reach the Dagster LXC
- Install-corpus-binary prerequisite: `rclone` on PATH (used by `everef` listing / the availability sensor; `ingest` does not need it)
- How-to guide: deploy the DB-VM — full VM with read-only Gold NFS mount, Postgres `eve` + Neo4j on localhost, eve-industry-serving loader, and the SSH trigger from the Dagster LXC (eve-industry-corpus phase 5)
- Project deployment doc for eve-industry-predict — MLflow-LXC + Dragonfly-LXC, dedicated UNAS `ML` share, proxied artefacts, and build order
- How-to guide: deploy the MLflow LXC — unprivileged container, 988 UID/GID map, `ML`-share NFS mount, own Postgres backing store, and `--serve-artifacts` proxy (eve-industry-predict phase 2)
- How-to guide: deploy the Dragonfly LXC — unprivileged container running the Feast online store bound to the LAN with a password and memory cap (eve-industry-predict phase 3)
- Static IPs for MLflow-LXC (192.168.2.213) and Dragonfly-LXC (192.168.2.214) in the network IP plan
- Hardware page for the DB-VM (Proxmox guest 212) — access, localhost-bound Postgres/Neo4j services, virtual specs, and SSH-tunnel note
- Hardware page for the Dagster-LXC (Proxmox guest 211) — access, Dagster/corpus services, the CORPUS_*/SERVING_* environment table, and virtual specs

### Changed

- eve-industry-corpus: UNAS Shared Drive named `EVE` (was assumed `corpus`); resolved the IP and share-name open items
- eve-industry-corpus phase 5 (DB-VM): Postgres on localhost with the serving loader co-located and SSH-triggered, rather than Postgres opened to the LXC subnet over `vmbr0`
- Orchestration repository renamed `eve-industry-orchestration` (was `eve-corpus-orchestration`) to match the `eve-industry-*` convention

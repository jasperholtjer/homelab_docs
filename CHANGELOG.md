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

### Changed

- eve-industry-corpus: UNAS Shared Drive named `EVE` (was assumed `corpus`); resolved the IP and share-name open items

# CLAUDE.md

Scope-specific rules for this repository. Defer to the global rules for everything
cross-cutting (git, markdown, doc-tone, language).

## Purpose

A living documentation repository for the homelab: hardware inventory, network
layout, and how-to guides. Optimized for quick recall of infrequently-used
knowledge — keep entries short, factual, and current.

## Layout

| Path                   | Holds                                                    |
| ---------------------- | -------------------------------------------------------- |
| `README.md`            | Index and overview table — the entry point               |
| `docs/network/`        | IP plan and topology                                     |
| `docs/hardware/`       | One file per device                                      |
| `docs/howto/`          | Task-oriented runbooks, one file per task                |
| `CHANGELOG.md`         | Every documentation change                               |

## Rules

- **One file per device** under `docs/hardware/`; one file per task under `docs/howto/`.
- **Filenames** are kebab-case and describe the device or task (e.g. `add-proxmox-vm.md`).
- **Cross-link** related pages with relative paths; never absolute or `file://`.
- **Single source of truth for addresses**: the IP plan in `docs/network/overview.md`.
  Device pages link to it rather than restating assignments.
- **Mark unknowns explicitly** as open items instead of omitting them, so gaps stay visible.
- **Record every change** in `CHANGELOG.md` under `[Unreleased]` in the same edit.

## Adding Content

- **New device**: create `docs/hardware/<device>.md`, add a row to the README overview
  table and the network IP plan, then changelog it.
- **New how-to**: create `docs/howto/<task>.md`, link it from `docs/howto/README.md`,
  then changelog it. Follow the guide template in `docs/howto/README.md`.

## Conventions

- All file content is English (per global rules), even though chat is Dutch.
- Tables for comparison, bullets for lists; fenced code blocks always carry a language tag.
- This repo is documentation only — no code, no build step.

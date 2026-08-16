# Network Overview

Subnet `192.168.2.0/24`, gateway `192.168.2.254` (KPN Box 12). Devices with a
service interface use a fixed IP so their web UIs stay reachable at a known address.

## IP Plan

| Device      | IP            | Assignment | Service                                              |
| ----------- | ------------- | ---------- | ---------------------------------------------------- |
| KPN Box 12  | 192.168.2.254 | Gateway    | Router / modem                                       |
| NUC         | 192.168.2.210 | Static     | Proxmox VE — [:8006](https://192.168.2.210:8006)     |
| Dagster-LXC | 192.168.2.211 | Static     | Dagster webserver — [:3000](http://192.168.2.211:3000) ([Dagster-LXC](../hardware/dagster-lxc.md), [eve-industry-corpus](../projects/eve-industry-corpus.md)) |
| DB-VM       | 192.168.2.212 | Static     | Postgres `eve` + Neo4j ([DB-VM](../hardware/db-vm.md), [eve-industry-corpus](../projects/eve-industry-corpus.md)) |
| MLflow-LXC  | 192.168.2.213 | Static     | MLflow tracking + registry — [:5000](http://192.168.2.213:5000) ([eve-industry-predict](../projects/eve-industry-predict.md)) |
| Dragonfly-LXC | 192.168.2.214 | Static   | Feast online store (Redis-compat) `:6379` ([eve-industry-predict](../projects/eve-industry-predict.md)) |
| UNAS 2      | 192.168.2.24  | Static     | UNAS web UI — [http](http://192.168.2.24/)           |

## Topology

```text
                  ┌──────────────────┐
                  │   KPN Box 12     │
                  │  192.168.2.254   │
                  └───┬──────────┬───┘
                wired │          │ WiFi
                      │          │
        ┌─────────────┴────┐  ┌──┴──────────┐
        │ Flex Mini 2.5G   │  │ Workstation │
        │ (USW-Flex-2.5G-5)│  │  Win 11     │
        └──┬────────────┬──┘  └─────────────┘
       P1  │            │  P4
     2.5G  │            │  2.5G
      ┌────┴────┐   ┌───┴───────┐
      │ UNAS 2  │   │   NUC     │
      │  24 TB  │   │ Proxmox 9 │
      └─────────┘   └───────────┘
```

- The Flex Mini switch uplinks to the KPN Box 12 (gateway `192.168.2.254`).
- Flex Mini ports: UNAS 2 on **port 1**, NUC on **port 4**.
- The workstation connects over WiFi to the KPN Box 12, not via the 2.5G switch.

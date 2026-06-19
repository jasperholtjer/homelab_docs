# NUC — Proxmox Host

Virtualization host running Proxmox VE 9.

## Access

- Web UI: [https://192.168.2.210:8006](https://192.168.2.210:8006)
- IP: `192.168.2.210` (static)

## Hardware

| Component | Spec                                                                  |
| --------- | --------------------------------------------------------------------- |
| Barebone  | ASUS NUC 15 Pro Kit (RNUC15CRHU500002)                                |
| CPU       | Intel Core Ultra 5 225H                                               |
| GPU       | Intel Arc Graphics (integrated)                                       |
| Memory    | Crucial 32 GB kit (2× 16 GB) DDR5-5600 CL46 SO-DIMM                    |
| Storage   | Samsung 990 PRO 2 TB — M.2 2280, PCIe 4.0 ×4, NVMe 2.0                 |
| Network   | WiFi 7 (onboard); wired uplink to Flex Mini 2.5G switch               |

The kit has 2× DDR5 SO-DIMM slots (both populated) and 2× M.2 slots (one populated),
leaving one M.2 slot free for expansion.

## Software

- Proxmox VE 9

## Reference

- [Barebone (RNUC15CRHU500002)](https://nl.nbb.com/nl/p/asus-nuc-15-pro-kit-rnuc15crhu500002-intel-core-ultra-5-225h-intel-arc-graphics-2x-ddr5-so-dimm-2x-m-2-wifi-7/a1081331)
- [SSD (Samsung 990 PRO 2 TB)](https://nl.nbb.com/nl/p/samsung-990-pro-ssd-2-tb-m-2-2280-pcie-4-0-x4-nvme-2-0-interne-solid-state-module/a-995339)
- [Memory (Crucial 32 GB DDR5-5600)](https://nl.nbb.com/nl/p/crucial-32gb-kit-2x16gb-ddr5-5600-cl46-so-dimm-geheugen/a1004273)

[🇷🇺 Русская версия](README.ru.md)

# OTUS Network Architect

> Labs and final project for the OTUS Network Architect course — Data Center fabric design using CLOS topology, VxLAN/EVPN Overlay on Cisco NX-OS / EVE-NG.

## Labs

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Address Space Design](01-Address-Space-Design/) | CLOS topology with 3 Spine + 4 Leaf. Address space distribution for Underlay network. |
| 02 | [Underlay — OSPF](02-Underlay-OSPF/) | OSPF configuration for DC Underlay network on Cisco NX-OS. |
| 03 | [Underlay — IS-IS](03-Underlay-ISIS/) | IS-IS configuration for DC Underlay network on Cisco NX-OS. |
| 04 | [Underlay — BGP](04-Underlay-BGP/) | eBGP configuration for DC Underlay with route-map and peer-templates. |
| 05 | [Multicast — PIM](05-Multicast-PIM/) | PIM Sparse-Mode and BSR configuration in DC topology. Multicast replication via EVE-NG. |
| 06 | [VxLAN Type 2 (L2 EVPN)](06-VxLAN-Type2/) | VxLAN EVPN Overlay for L2 connectivity. Spine as Route Reflector. |
| 07 | [VxLAN Route (L3 EVPN)](07-VxLAN-Route/) | L3 routing between clients in Overlay. Per-client VNI, VPC pair configuration. |
| 08 | [VxLAN Multipod](08-VxLAN-Multipod/) | L2/L3 connectivity between two pods via Multipod VxLAN EVPN. |
| 09 | [Project](09-Project/) | Final project: VxLAN Control Plane migration from Multicast to EVPN with Multipod support. |

## Environment

- Simulator: [EVE-NG](https://www.eve-ng.net/)
- Platform: Cisco NX-OS
- Course: [OTUS Network Architect](https://otus.ru/lessons/network-architect/)

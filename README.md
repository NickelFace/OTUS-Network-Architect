[🇷🇺 Русская версия](README.ru.md)

![NX-OS](https://img.shields.io/badge/Cisco_NX--OS-DC_Fabric-1BA0D7?logo=cisco&logoColor=white)
![VxLAN](https://img.shields.io/badge/VxLAN-EVPN-4A90D9)
![EVE-NG](https://img.shields.io/badge/EVE--NG-Simulation-2E86AB)
![Status](https://img.shields.io/badge/status-completed-brightgreen)

# Data Centre Fabric Design — VxLAN / EVPN on Cisco NX-OS

End-to-end design and implementation of a CLOS data-centre fabric: address space planning, three underlay routing protocols, Multicast replication, and a full VxLAN EVPN overlay (L2 + L3 + Multipod). Completed as a graduation project of the OTUS Network Architect programme, but the design decisions and troubleshooting are my own.

## What was built

```
      Spine-1   Spine-2   Spine-3
         │    ╲  │  ╱    │
      Leaf-1  Leaf-2  Leaf-3  Leaf-4
    (VTEP)   (VTEP)  (VTEP)  (VTEP)
```

3 Spines · 4 Leaves · full-mesh eBGP underlay · VxLAN EVPN overlay  
Simulator: EVE-NG · Platform: Cisco NX-OS

## Lab progression

| # | Topic | What was decided / solved |
|---|-------|--------------------------|
| 01 | [Address Space Design](01-Address-Space-Design/) | Carved /30 point-to-point links for Underlay and loopback pool for VTEP tunnel sources. |
| 02 | [Underlay — OSPF](02-Underlay-OSPF/) | Deployed OSPF as Underlay baseline; compared convergence behaviour vs IS-IS. |
| 03 | [Underlay — IS-IS](03-Underlay-ISIS/) | Replaced OSPF with IS-IS; evaluated TLV efficiency on NX-OS in pure L2 domain. |
| 04 | [Underlay — BGP](04-Underlay-BGP/) | Selected eBGP as final Underlay protocol; applied route-map and peer-templates to scale config. |
| 05 | [Multicast — PIM](05-Multicast-PIM/) | Configured PIM Sparse-Mode + BSR for BUM traffic replication before switching to EVPN. |
| 06 | [VxLAN Type 2 — L2 EVPN](06-VxLAN-Type2/) | Extended L2 segments across Leaves via EVPN MAC routes; Spines as Route Reflectors. |
| 07 | [VxLAN Route — L3 EVPN](07-VxLAN-Route/) | Enabled inter-tenant routing in Overlay; per-client L3 VNI, VPC peer-link for redundancy. |
| 08 | [VxLAN Multipod](08-VxLAN-Multipod/) | Stretched L2/L3 between two pods over a Multipod inter-pod link with dedicated DCI VNI. |
| 09 | [Graduation Project](09-Project/) | Migrated control plane from PIM/Multicast to EVPN; maintained Multipod continuity during cut-over. |

## Key design choices

| Decision | Rationale |
|---|---|
| eBGP as Underlay (not OSPF/IS-IS) | Scales to large fabrics without area/level constraints; each Leaf is its own AS |
| Spine as EVPN Route Reflector | Avoids full-mesh iBGP between Leaves; single point of policy |
| Per-client L3 VNI | Clean tenant separation; simplifies security policy attachment |
| EVPN over PIM for BUM | Eliminates multicast dependency; ingress replication is deterministic |

## Environment

- Simulator: [EVE-NG](https://www.eve-ng.net/)
- Platform: Cisco NX-OS (7.x / 9.x)
- Related: [OTUS-Network-Engineer](https://github.com/NickelFace/OTUS-Network-Engineer) — multi-site WAN (BGP / OSPF / DMVPN)

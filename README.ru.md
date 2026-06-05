[🇬🇧 English version](README.md)

![NX-OS](https://img.shields.io/badge/Cisco_NX--OS-DC_Fabric-1BA0D7?logo=cisco&logoColor=white)
![VxLAN](https://img.shields.io/badge/VxLAN-EVPN-4A90D9)
![EVE-NG](https://img.shields.io/badge/EVE--NG-Simulation-2E86AB)
![Status](https://img.shields.io/badge/status-завершён-brightgreen)

# Проектирование фабрики ЦОД — VxLAN / EVPN на Cisco NX-OS

Сквозное проектирование и реализация CLOS-фабрики ЦОД: планирование адресного пространства, три протокола Underlay, мультикастовая репликация, полноценный VxLAN EVPN Overlay (L2 + L3 + Multipod). Выполнено в рамках программы OTUS «Сетевой Архитектор», проектные решения и траблшутинг — собственные.

## Что было построено

```
      Spine-1   Spine-2   Spine-3
         │    ╲  │  ╱    │
      Leaf-1  Leaf-2  Leaf-3  Leaf-4
    (VTEP)   (VTEP)  (VTEP)  (VTEP)
```

3 Spine · 4 Leaf · full-mesh eBGP Underlay · VxLAN EVPN Overlay  
Симулятор: EVE-NG · Платформа: Cisco NX-OS

## Лабораторные работы

| # | Тема | Что решалось / какое решение принято |
|---|------|--------------------------------------|
| 01 | [Проектирование адресного пространства](01-Address-Space-Design/) | Нарезка /30 для point-to-point линков Underlay, пул loopback-адресов для VTEP. |
| 02 | [Underlay — OSPF](02-Underlay-OSPF/) | Развёрнут OSPF как базовый Underlay; сравнение поведения сходимости с IS-IS. |
| 03 | [Underlay — IS-IS](03-Underlay-ISIS/) | IS-IS вместо OSPF; оценка эффективности TLV на NX-OS в чистом L2-домене. |
| 04 | [Underlay — BGP](04-Underlay-BGP/) | Выбран eBGP как итоговый протокол Underlay; route-map и peer-templates для масштабируемости. |
| 05 | [Multicast — PIM](05-Multicast-PIM/) | PIM Sparse-Mode + BSR для репликации BUM-трафика до перехода на EVPN. |
| 06 | [VxLAN Type 2 — L2 EVPN](06-VxLAN-Type2/) | Растянуты L2-сегменты между Leaf через EVPN MAC-маршруты; Spine — Route Reflector. |
| 07 | [VxLAN Route — L3 EVPN](07-VxLAN-Route/) | Маршрутизация между тенантами в Overlay; per-client L3 VNI, VPC-пара для резервирования. |
| 08 | [VxLAN Multipod](08-VxLAN-Multipod/) | L2/L3-связность между двумя pod через Multipod с выделенным DCI VNI. |
| 09 | [Дипломный проект](09-Project/) | Миграция Control Plane с PIM/Multicast на EVPN без потери Multipod-связности. |

## Ключевые проектные решения

| Решение | Обоснование |
|---|---|
| eBGP в Underlay (не OSPF/IS-IS) | Масштабируется без ограничений зон/уровней; каждый Leaf — отдельная AS |
| Spine как EVPN Route Reflector | Исключает full-mesh iBGP между Leaf; единая точка политики |
| Per-client L3 VNI | Чистое разделение тенантов; упрощает привязку политик безопасности |
| EVPN вместо PIM для BUM | Убирает зависимость от мультикаста; ingress replication детерминирован |

## Среда

- Симулятор: [EVE-NG](https://www.eve-ng.net/)
- Платформа: Cisco NX-OS (7.x / 9.x)
- Смежный репо: [OTUS-Network-Engineer](https://github.com/NickelFace/OTUS-Network-Engineer) — мультисайтовая WAN (BGP / OSPF / DMVPN)

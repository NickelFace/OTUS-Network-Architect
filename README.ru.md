[🇬🇧 English version](README.md)

# OTUS Сетевой Архитектор

> Лабораторные работы и финальный проект курса OTUS «Сетевой Архитектор» — проектирование фабрики ЦОД на базе топологии CLOS, VxLAN/EVPN Overlay на Cisco NX-OS / EVE-NG.

## Лабораторные работы

| # | Тема | Описание |
|---|------|----------|
| 01 | [Проектирование адресного пространства](01-Address-Space-Design/) | Топология CLOS: 3 Spine + 4 Leaf. Распределение адресного пространства для Underlay-сети. |
| 02 | [Underlay — OSPF](02-Underlay-OSPF/) | Настройка OSPF для Underlay-сети ЦОД на Cisco NX-OS. |
| 03 | [Underlay — IS-IS](03-Underlay-ISIS/) | Настройка IS-IS для Underlay-сети ЦОД на Cisco NX-OS. |
| 04 | [Underlay — BGP](04-Underlay-BGP/) | Настройка eBGP для Underlay с route-map и peer-templates. |
| 05 | [Multicast — PIM](05-Multicast-PIM/) | PIM Sparse-Mode и BSR в топологии ЦОД. Мультикастовая репликация через EVE-NG. |
| 06 | [VxLAN Type 2 (L2 EVPN)](06-VxLAN-Type2/) | VxLAN EVPN Overlay для L2-связности. Spine в роли Route Reflector. |
| 07 | [VxLAN Route (L3 EVPN)](07-VxLAN-Route/) | L3-маршрутизация между клиентами в Overlay. Per-client VNI, настройка VPC-пары. |
| 08 | [VxLAN Multipod](08-VxLAN-Multipod/) | L2/L3-связность между двумя pod через Multipod VxLAN EVPN. |
| 09 | [Проект](09-Project/) | Финальный проект: миграция Control Plane VxLAN с Multicast на EVPN с поддержкой Multipod. |

## Среда

- Симулятор: [EVE-NG](https://www.eve-ng.net/)
- Платформа: Cisco NX-OS
- Курс: [OTUS Сетевой Архитектор](https://otus.ru/lessons/network-architect/)

[🇬🇧 English version](README.md)

# OTUS Сетевой Архитектор

> Лабораторные работы и финальный проект курса OTUS «Сетевой Архитектор» — проектирование фабрики ЦОД на базе топологии CLOS, VxLAN/EVPN Overlay на Cisco NX-OS / EVE-NG.

## Лабораторные работы

| # | Тема | Папка | Описание |
|---|------|-------|----------|
| 01 | Проектирование адресного пространства | [1.Address space design](1.Address%20space%20design/) | Топология CLOS: 3 Spine + 4 Leaf. Распределение адресного пространства для Underlay-сети. |
| 02 | Underlay — OSPF | [2.Overlay OSPF](2.Overlay%20OSPF/) | Настройка OSPF для Underlay-сети ЦОД на Cisco NX-OS. |
| 03 | Underlay — IS-IS | [3.Underlay ISIS](3.Underlay%20ISIS/) | Настройка IS-IS для Underlay-сети ЦОД на Cisco NX-OS. |
| 04 | Underlay — BGP | [4.Underlay BGP](4.Underlay%20BGP/) | Настройка eBGP для Underlay с route-map и peer-templates. |
| 05 | Multicast — PIM | [5.Multicast PIM](5.Multicast%20PIM/) | PIM Sparse-Mode и BSR в топологии ЦОД. Мультикастовая репликация через EVE-NG. |
| 06 | VxLAN Type 2 (L2 EVPN) | [6. VxLAN type 2](6.%20VxLAN%20type%202/) | VxLAN EVPN Overlay для L2-связности. Spine в роли Route Reflector. |
| 07 | VxLAN Route (L3 EVPN) | [7.VxLAN Route](7.VxLAN%20%20Route/) | L3-маршрутизация между клиентами в Overlay. Per-client VNI, настройка VPC-пары. |
| 08 | VxLAN Multipod | [8.VxLAN. Multipod](8.VxLAN.%20Multipod/) | L2/L3-связность между двумя pod через Multipod VxLAN EVPN. |
| 09 | Проект | [9.Project work](9.Project%20work/) | Финальный проект: миграция Control Plane VxLAN с Multicast на EVPN с поддержкой Multipod. |

## Среда

- Симулятор: [EVE-NG](https://www.eve-ng.net/)
- Платформа: Cisco NX-OS
- Курс: [OTUS Сетевой Архитектор](https://otus.ru/lessons/network-architect/)

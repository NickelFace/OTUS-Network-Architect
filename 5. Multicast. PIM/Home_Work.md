# Multicast. PIM

Цель:

Настроить PIM в сети.

В этой работе мы ожидаем, что вы самостоятельно:

1. Настроите PIM на всех устройствах (кроме коммутаторов доступа);

  *Для IP связанности между устройствами можно использовать любой протокол динамической маршрутизации; 2. План работы, адресное пространство, схема сети, настройки - зафиксируете в документации;

Дополнительно предлагается реализовать Multicast совместно с VxLAN(к этой части самостоятельной работы возможно вернуться позже)



За основу взял схему из раздела [OSPF](https://github.com/NickelFace/OTUS-Network-Architect/blob/main/2.Overlay_OSPF/Home_Work.md), где указал логику построения сети ЦОД, а  теперь требуется организовать  мультикаст в данной топологии.

Так как в интернете довольно мало информации на данную тематику,а [некоторые статьи](https://linkmeup.ru/blog/1204/)  не подойдут для EVE-NG, то придется организовать свою статью. Первая проблема с которой столкнулся, это как организовать источник мультикаста, а заодно и клиента. За решение данного вопроса я воспользовалься [инструкцией](https://www.eve-ng.net/index.php/documentation/howtos/howto-save-your-settings-to-be-as-default-on-qemu-node/) по созданию своего [образа](https://disk.yandex.ru/d/_UKl3leYfNVqGA). Для организации сервера мне потребовался пакет [tstools](https://onstartup.ru/utility/tstools/), а для организации клиента [smcroute](https://onstartup.ru/set/smcroute/). 



![](./img/Schema1.png)

**Настройка NEXUS:**

```
 <details>
<summary>NXOS1</summary>
<pre><code>
conf t
! 
hostname NX1
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
router ospf 1
  router-id 1.1.1.1
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.1
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.10.11.254/24
  ip ospf passive-interface
  ip router ospf 1 area 0.0.0.1
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.1/24
  ip router ospf 1 area 0.0.0.1
!
end
copy run star 
</code></pre>
</details>
```

```
<details>
<summary>NXOS2</summary>
<pre><code>
conf t
!
hostname NX2
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
router ospf 1
  router-id 1.1.1.2
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/4
  no switchport
  ip address 10.15.0.6/31
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.2/24
  ip router ospf 1 area 0.0.0.0
!
end
copy run star
</code></pre>
</details>
```

```
<details>
  <summary>NXOS3</summary>
<pre><code>
 conf t
!
hostname NX3
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
router ospf 1
  router-id 1.1.1.3
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/4
  no switchport
  ip address 10.15.1.6/31
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.3/24
  ip router ospf 1 area 0.0.0.0
!
end
copy run star
</code></pre>
</details>
```

```
<details>
  <summary>NXOS4</summary>
<pre><code>
conf t
!
hostname NX4
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
router ospf 1
  router-id 1.1.1.4
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.1
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.16.0.0/31
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.1
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.4/24
  ip router ospf 1 area 0.0.0.1
!
end
copy run star
</code></pre>
</details>
```

```
<details>
<summary>NXOS5</summary>
<pre><code>
conf t
!
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
hostname NX5
!
router ospf 1
  router-id 1.1.1.5
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  ip address 10.10.12.2/24
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  ip pim dr-priority 1000
  no shutdown
!
interface Ethernet1/4
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.5/24
  ip router ospf 1 area 0.0.0.0
!
end
copy run star
 </code></pre>
</details>
```

```
<details>
<summary>NXOS6</summary>
<pre><code>
conf t
!
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
hostname NX6
!
router ospf 1
  router-id 1.1.1.6
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  ip address 10.10.10.254/24
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.6/24
  ip router ospf 1 area 0.0.0.0
!
end
copy run star
 </code></pre>
</details>
```

```
<details>
<summary>NXOS7</summary>
<pre><code>
conf t
!
hostname NX7
!
feature ospf
feature pim
!
ip pim rp-address 1.1.1.11
!
router ospf 1
  router-id 1.1.1.7
  passive-interface default
!
interface Ethernet1/1
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  medium p2p
  ip unnumbered loopback0
  ip ospf authentication-key 3 OTUS
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/4
  no switchport
  ip address 10.10.12.1/24
  ip router ospf 1 area 0.0.0.0
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.7/24
  ip router ospf 1 area 0.0.0.0
!
end
copy run star
</code></pre>
</details>
```

```
<details>
<summary>R11</summary>
<pre><code>
enable
configure terminal
!
hostname R11
line con 0
exec-t 0 0
exit
no ip domain loo
!
router ospf 1
router-id 1.1.1.11
!
interface Ethernet0/0
 ip address 10.15.0.7 255.255.255.254
 ip pim sparse-mode
 ip ospf authentication-key OTUS
 ip ospf network point-to-point
 ip ospf 1 area 0
!
interface Ethernet0/1
 ip address 10.15.1.7 255.255.255.254
 ip pim sparse-mode
 ip ospf authentication-key OTUS
 ip ospf network point-to-point
 ip ospf 1 area 0
!
interface Ethernet0/2
 ip address 10.16.0.1 255.255.255.254
 ip pim sparse-mode
 ip ospf authentication-key OTUS
 ip ospf network point-to-point
 ip ospf 1 area 1
!
interface Loopback0
 ip address 1.1.1.11 255.255.255.0
 ip pim sparse-mode
 ip ospf 1 area 0
!
ip multicast-routing 
ip pim rp-address 1.1.1.11
! 
end
wr
</code></pre>
</details>
```

Настройка Source Multicast

<details>
<summary>Server</summary>
cat /etc/network/interfaces/
<pre><code>
auto ens3
iface ens3  inet static
        address 10.10.10.2
        netmask 255.255.255.0
        gateway 10.10.10.1
</code></pre>
Запуск источника выполняется командой:
<code> 
tsplay ./video.ts 239.0.0.100:1234 -loop -i 10.10.10.2 &
</code>
</details>

Настройка клиентов:

<details>
<summary>Client13</summary>
cat /etc/network/interfaces/
<pre><code>
auto ens3
iface ens3 inet static
        address 10.10.12.10
        netmask 255.255.255.0
        gateway 10.10.12.254
</code></pre>
Запуск подписки на мультикаст рассылку выполняется командой:
<code> 
smcroute -j ens3 239.0.0.100
</code>
</details>

<details>
<summary>Client14</summary>
cat /etc/network/interfaces/
<pre><code>
auto ens3
iface ens3 inet static
        address 10.10.11.1
        netmask 255.255.255.0
        gateway 10.10.11.254
</code></pre>
Запуск подписки на мультикаст рассылку выполняется командой:
<code> 
smcroute -j ens3 239.0.0.100
</code>
</details>

А устройства SW9, SW10, SW11 выполняют просто функцию коммутатора.

<details>
<summary>SW9</summary>
<pre><code>
enable
configure terminal
!
ip multicast-routing 
!
no ip igmp snooping vlan 100
!
hostname SW9
line con 0
exec-t 0 0
exit
no ip domain loo
!
interface Ethernet0/0
 switchport access vlan 100
 switchport mode access
 spanning-tree bpdufilter enable
!
interface Ethernet0/1
 switchport access vlan 100
 switchport mode access
 spanning-tree bpdufilter enable
!
interface Vlan100
 ip address 10.10.10.1 255.255.255.0
 ip pim sparse-mode
!
ip route 0.0.0.0 0.0.0.0 10.10.10.254
end
wr
</code></pre>
</details> 

<details>
<summary>SW10</summary>
<pre><code>
enable
configure terminal
!
hostname SW10
line con 0
exec-t 0 0
exit
no ip domain loo
interface Ethernet0/0
 switchport access vlan 100
 switchport mode access
 duplex full
!
interface Ethernet0/1
 switchport access vlan 100
 switchport mode access
 duplex full
!
interface Ethernet0/2
 switchport access vlan 100
 switchport mode access
 duplex full
!
interface Vlan100
 ip address 10.10.12.254 255.255.255.0
 ip pim sparse-mode
!
ip sla 1
 icmp-echo 10.10.12.2 source-interface Vlan100
 frequency 10
ip sla schedule 1 life forever start-time now
!
ip route 0.0.0.0 0.0.0.0 10.10.12.2 track 1
ip route 0.0.0.0 0.0.0.0 10.10.12.1
!
end
wr
</code></pre>
</details> 

<details>
<summary>SW11</summary>
<pre><code>
enable
configure terminal
!
ip multicast-routing
!
hostname SW11
line con 0
exec-t 0 0
exit
no ip domain loo
!
interface Ethernet0/0
 switchport access vlan 100
 switchport mode access
 duplex full
 spanning-tree bpdufilter enable
!
interface Ethernet0/1
 switchport access vlan 100
 switchport mode access
 duplex full
 spanning-tree bpdufilter enable
!         
interface Vlan100
 ip address 10.10.11.1 255.255.255.0
 ip pim sparse-mode
!
ip route 0.0.0.0 0.0.0.0 10.10.11.254
!
end
wr
</code></pre>
</details> 

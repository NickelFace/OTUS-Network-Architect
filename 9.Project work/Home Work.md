# VxLAN. Миграция Control Plane c Multicast на EVPN

Цель:

Составить план перехода от CP Multicast к EVPN

План работы:

1. Настроить схему сети для VxLAN CP Multicast.
2. Предоставить план перехода на EVPN.
3. Настроить схему сети для VxLAN Multipod.

![Scheme](./img/Scheme.png)

## Настроить схему сети для VxLAN CP Multicast.

Описание:

На данный момент буду приводить конфигурацию VxLAN через мультикаст, то есть передача информации в сети об определенном VNI будет распространяться через многоадресную рассылку в режиме **sparse mode**. В данном примере мне потребуется решить следующие задачи: кто будет участвовать в качестве RP(и как получить эту информацию), многоадресная IP адресация(чтобы по сети не гуляли одинаковые mac-адреса) и так далее. 

Что не получилось:

Для данной схемы получилось настроить только L2 связь, так как при настройке L3 получил сообщение:

```
TRM not supported on this platform
```

А также в образах отсутствует поддержка bfd(команды есть, но самих пакетов нет) 

**Приступим к настройке NEXUS:**

<details>
  <summary>NXOS1</summary>
<pre><code>
configure terminal
!
hostname NX1
!
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
!
no ip domain-lookup
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr listen
vlan 1,11
vlan 11
  vn-segment 100200
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.1/32 
ip prefix-list LOOPBACK seq 10 permit 10.1.1.1/32 
ip prefix-list P2P seq 5 permit 10.16.0.2/31 
ip prefix-list P2P seq 10 permit 172.16.2.0/31 
route-map BGP-OUT permit 10
  match ip address prefix-list LOOPBACK P2P 
!
interface Vlan11
  no shutdown
  ip address 172.16.2.254/24
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
!
interface nve1
  no shutdown
  source-interface loopback0
  member vni 100200 mcast-group 231.1.2.1
!
interface Ethernet1/1
  no switchport
  ip address 10.16.0.3/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  switchport mode trunk
!
interface loopback0
  ip address 1.1.1.1/32
  ip address 10.1.1.1/32 secondary
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
boot nxos bootflash:/nxos.9.2.2.bin 
!
router bgp 64551
  router-id 1.1.1.1
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
  template peer NXOS4
    remote-as 64554
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
  neighbor 10.16.0.2
    inherit peer NXOS4
!
end
wr
</code></pre>
</details>
 <details>
<summary>NXOS2</summary>
<pre><code>
configure terminal
hostname NX2
!
feature bgp
feature pim
!
ip pim bsr bsr-candidate loopback1 priority 90
ip pim bsr rp-candidate loopback1 group-list 224.0.0.0/4 priority 90
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr forward listen
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.2/32 
ip prefix-list LOOPBACK seq 10 permit 10.12.10.1/32 
ip prefix-list P2P seq 5 permit 10.15.0.0/31 
ip prefix-list P2P seq 10 permit 10.15.0.2/31 
ip prefix-list P2P seq 15 permit 10.15.0.4/31 
ip prefix-list P2P seq 20 permit 10.15.0.6/31 
route-map BGP-OUT permit 10
  match ip address prefix-list LOOPBACK P2P 
!
interface Ethernet1/1
  no switchport
  ip address 10.15.0.0/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.15.0.2/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  ip address 10.15.0.4/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/4
  no switchport
  ip address 10.15.0.6/31
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.2/32
  ip pim sparse-mode
!
interface loopback1
  ip address 10.12.10.1/32
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
boot nxos bootflash:/nxos.9.2.2.bin 
!
router bgp 64552
  router-id 1.1.1.2
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
!
  template peer LEAF_VPC
    remote-as 64555
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  template peer NXOS6
    remote-as 64556
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  template peer R11
    remote-as 64777
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  neighbor 10.15.0.1
    inherit peer NXOS6
  neighbor 10.15.0.3
    inherit peer LEAF_VPC
  neighbor 10.15.0.5
    inherit peer LEAF_VPC
  neighbor 10.15.0.7
    inherit peer R11
end
wr
</code></pre>
</details>
<details>
  <summary>NXOS3</summary>
<pre><code>
configure terminal
hostname NX3
!
nv overlay evpn
feature bgp
feature pim
feature nv overlay
!
no ip domain-lookup
!
ip pim bsr bsr-candidate loopback0 priority 90
ip pim bsr rp-candidate loopback0 group-list 224.0.0.0/4 priority 90
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr forward listen
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.3/32 
ip prefix-list P2P permit 10.15.1.0/31
ip prefix-list P2P permit 10.15.1.2/31 
ip prefix-list P2P permit 10.15.1.4/31 
ip prefix-list P2P permit 10.15.1.6/31 
route-map BGP-OUT permit 10
  match ip address prefix-list LOOPBACK P2P 
!
interface Ethernet1/1
  no switchport
  ip address 10.15.1.0/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.15.1.2/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  ip address 10.15.1.4/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/4
  no switchport
  ip address 10.15.1.6/31
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.3/32
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
router bgp 64552
  router-id 1.1.1.3
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
!
  template peer LEAF_VPC
    remote-as 64555
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  template peer NXOS6
    remote-as 64556
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  template peer R11
    remote-as 64777
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  neighbor 10.15.1.1
    inherit peer NXOS6
  neighbor 10.15.1.3
    inherit peer LEAF_VPC
  neighbor 10.15.1.5
    inherit peer LEAF_VPC
  neighbor 10.15.1.7
    inherit peer R11
end
wr
</code></pre>
</details>
<details>
<summary>NXOS4</summary>
<pre><code>
configure terminal
hostname NX4
!
feature bgp
feature pim
!
ip pim bsr bsr-candidate loopback0 priority 90
ip pim bsr rp-candidate loopback0 group-list 224.0.0.0/4 priority 90
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr forward listen
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.4/32 
ip prefix-list P2P seq 5 permit 10.16.0.2/31 
ip prefix-list P2P seq 10 permit 10.16.0.0/31 
route-map BGP-OUT permit 10
  match ip address prefix-list LOOPBACK P2P 
!
interface Ethernet1/1
  no switchport
  ip address 10.16.0.2/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.16.0.0/31
  ip pim sparse-mode
  no shutdown
!
interface loopback0
  ip address 1.1.1.4/32
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
router bgp 64554
  router-id 1.1.1.4
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
!
  template peer NXOS1
    remote-as 64551
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
  template peer R11
    remote-as 64777
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
  neighbor 10.16.0.1
    inherit peer R11
  neighbor 10.16.0.3
    inherit peer NXOS1
!
end
wr
</code></pre>
</details>
<details>
<summary>NXOS5</summary>
<pre><code>
configure terminal
hostname NX5
!
cfs eth distribute
nv overlay evpn
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature nv overlay
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr listen
vlan 1,10
vlan 10
  vn-segment 10010
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.5/32 
ip prefix-list LOOPBACK seq 10 permit 10.1.1.5/32 
ip prefix-list P2P seq 5 permit 10.15.0.4/31 
ip prefix-list P2P seq 10 permit 10.15.1.4/31 
ip prefix-list P2P seq 15 permit 10.15.2.0/31 
!
vrf context VPC
vpc domain 1
  peer-keepalive destination 10.15.2.0 source 10.15.2.1 vrf VPC
!
interface Vlan10
  no shutdown
  ip address 172.16.10.254/24
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
!
interface port-channel1
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link
!
interface port-channel2
  switchport mode trunk
  vpc 1
!
interface nve1
  no shutdown
  source-interface loopback0
  member vni 10010 mcast-group 230.1.1.1
!
interface Ethernet1/1
  no switchport
  ip address 10.15.0.5/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.15.1.5/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  vrf member VPC
  ip address 10.15.2.1/31
  no shutdown
!
interface Ethernet1/4
  switchport mode trunk
  channel-group 1 mode active
!
interface Ethernet1/5
  switchport mode trunk
  channel-group 1 mode active
!
interface Ethernet1/6
  switchport mode trunk
  spanning-tree bpdufilter enable
  channel-group 2 mode active
!
interface loopback0
  ip address 1.1.1.5/32
  ip address 10.1.1.5/32 secondary
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
router bgp 64555
  router-id 1.1.1.5
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
!
  template peer SPINE
    remote-as 64552
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  neighbor 10.15.0.4
    inherit peer SPINE
  neighbor 10.15.1.4
    inherit peer SPINE
end
wr
</code></pre>
</details>
<details>
<summary>NXOS6</summary>
<pre><code>
configure terminal
hostname NX6
!
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
!
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr listen
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
!
vlan 1,10-11
vlan 10
  vn-segment 10010
vlan 11
  vn-segment 100200
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.6/32 
ip prefix-list LOOPBACK seq 10 permit 10.1.1.6/32 
ip prefix-list P2P seq 5 permit 10.15.0.0/31 
!
route-map BGP-OUT permit 10
  match ip address prefix-list LOOPBACK P2P 
!
interface Vlan10
  no shutdown
  ip address 172.16.10.253/24
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
!
interface Vlan11
  no shutdown
  ip address 172.16.2.253/24
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
!
interface nve1
  no shutdown
  source-interface loopback0
  member vni 10010 mcast-group 230.1.1.1
  member vni 100200 mcast-group 231.1.2.1
!
interface Ethernet1/1
  no switchport
  ip address 10.15.0.1/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.15.1.1/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  switchport mode trunk
!
interface loopback0
  ip address 1.1.1.6/32
  ip address 10.1.1.6/32 secondary
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
boot nxos bootflash:/nxos.9.2.2.bin 
router bgp 64556
  router-id 1.1.1.6
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
!
  template peer SPINE
    remote-as 64552
    log-neighbor-changes
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
!
  neighbor 10.15.0.0
    inherit peer SPINE
  neighbor 10.15.1.0
    inherit peer SPINE
end
wr
</code></pre>
</details>
<details>
<summary>NXOS7</summary>
<pre><code>
configure terminal
hostname NX7
!
cfs eth distribute
nv overlay evpn
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature lacp
feature vpc
feature nv overlay
!
fabric forwarding anycast-gateway-mac 0001.0001.0001
ip pim log-neighbor-changes
ip pim ssm range 232.0.0.0/8
ip pim bsr listen
vlan 1,10
vlan 10
  vn-segment 10010
!
ip prefix-list LOOPBACK seq 5 permit 1.1.1.7/32 
ip prefix-list LOOPBACK seq 10 permit 10.1.1.5/32 
ip prefix-list P2P seq 5 permit 10.15.0.2/31 
route-map BGP-OUT permit 10
  match ip address prefix-list LOOPBACK P2P 
!
vrf context VPC
vpc domain 1
  peer-keepalive destination 10.15.2.1 source 10.15.2.0 vrf VPC
!
interface Vlan10
  no shutdown
  ip address 172.16.10.254/24
  ip pim sparse-mode
  fabric forwarding mode anycast-gateway
!
interface port-channel1
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link
!
interface port-channel2
  switchport mode trunk
  vpc 1
!
interface nve1
  no shutdown
  source-interface loopback0
  member vni 10010 mcast-group 230.1.1.1
!
interface Ethernet1/1
  no switchport
  ip address 10.15.0.3/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/2
  no switchport
  ip address 10.15.1.3/31
  ip pim sparse-mode
  no shutdown
!
interface Ethernet1/3
  no switchport
  vrf member VPC
  ip address 10.15.2.0/31
  no shutdown
!
interface Ethernet1/4
  switchport mode trunk
  channel-group 1 mode active
!
interface Ethernet1/5
  switchport mode trunk
  channel-group 1 mode active
!
interface Ethernet1/6
  switchport mode trunk
  spanning-tree bpdufilter enable
  channel-group 2 mode active
!
interface loopback0
  ip address 1.1.1.7/32
  ip address 10.1.1.5/32 secondary
  ip pim sparse-mode
!
cli alias name wr copy running-config startup-config
line console
  exec-timeout 0
line vty
  exec-timeout 0
!
router bgp 64555
  router-id 1.1.1.7
  bestpath as-path multipath-relax
  address-family ipv4 unicast
    redistribute direct route-map BGP-OUT
    maximum-paths 4
!
  template peer SPINE
    remote-as 64552
    password 3 9125d59c18a9b015
    address-family ipv4 unicast
  neighbor 10.15.0.2
    inherit peer SPINE
  neighbor 10.15.1.2
    inherit peer SPINE
end
wr
</code></pre>
</details>

**Настройка Switch:**

<details>
  <summary>SW9</summary>
<pre><code>
enable
configure terminal
!
hostname SW9
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 spanning-tree bpdufilter enable
!
interface Vlan10
 ip address 172.16.10.1 255.255.255.0
!
interface Vlan11
 ip address 172.16.2.2 255.255.255.0
!
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
!
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 spanning-tree bpdufilter enable
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 spanning-tree bpdufilter enable
!
interface Ethernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 spanning-tree bpdufilter enable
!
interface Vlan10
 ip address 172.16.10.20 255.255.255.0
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
hostname SW11
!
interface Ethernet0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Vlan11
 ip address 172.16.2.1 255.255.255.0
!
end
wr
</code></pre>
</details>
А теперь покажу 2 простроенных туннеля nve и вывод устройств:
<details>
<summary>NX1</summary>
<pre><code>
NX1# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      10.1.1.6         Up    DP        02:58:21 n/a     
</code></pre>
</details>
<details>
<summary>NX5-NX7(VPC пара)</summary>
<pre><code>
NX5(config-if)# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      10.1.1.6         Up    DP        06:01:34 n/a  
</code></pre>
</details>
<details>
<summary>NX6</summary>
<pre><code>
NX6(config-if)# end
NX6# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      10.1.1.1         Up    DP        02:56:01 n/a              
nve1      10.1.1.5         Up    DP        03:32:54 n/a       
</code></pre>
</details>
Чтобы не загромождать проект лишними выводами, вставлю далее картинку с указанием построенного пути:

![Scheme2](./img/Scheme2.png)

Multicast на данный момент считается уже устаревшим решением, поэтому главной задачей данного проекта перевести сеть на EVPN

## Предоставить план перехода на EVPN.




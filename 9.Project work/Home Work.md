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




























Для начала проверим пиринг:


<details>
<summary>NXOS1</summary>
<pre><code>
NX1#  sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.1, local AS number 64300
BGP table version is 563, L2VPN EVPN config peers 1, capable peers 1
11 network entries and 11 paths using 2420 bytes of memory
BGP attribute entries [6/984], BGP AS path entries [2/28]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.4         4 64550    3734     887      563    0    0 11:06:33 6     
</code></pre>
</details>
<details>
<summary>NXOS2</summary>
<pre><code>
NX2# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.2, local AS number 65000
BGP table version is 331, L2VPN EVPN config peers 4, capable peers 4
41 network entries and 41 paths using 9020 bytes of memory
BGP attribute entries [18/2952], BGP AS path entries [3/22]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.4         4 64550     100     158      331    0    0 00:57:18 5         
1.1.1.5         4 65200      85     119      331    0    0 00:57:17 10        
1.1.1.6         4 65100     182     125      331    0    0 00:57:19 15        
1.1.1.7         4 65200     142     119      331    0    0 00:57:17 11        
NX2#      
</code></pre>
</details>
<details>
<summary>NXOS3</summary>
<pre><code>
NX3# sh bgp l2vpn evpn summary
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.3, local AS number 65000
BGP table version is 331, L2VPN EVPN config peers 4, capable peers 4
41 network entries and 41 paths using 9020 bytes of memory
BGP attribute entries [18/2952], BGP AS path entries [3/22]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.4         4 64550      99     158      331    0    0 00:57:12 5         
1.1.1.5         4 65200      86     119      331    0    0 00:57:13 10        
1.1.1.6         4 65100     182     124      331    0    0 00:57:10 15        
1.1.1.7         4 65200     141     119      331    0    0 00:57:13 11
</code></pre>
</details>
<details>
<summary>NXOS4</summary>
<pre><code>
NX4# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.4, local AS number 64550
BGP table version is 5676, L2VPN EVPN config peers 3, capable peers 3
41 network entries and 77 paths using 13484 bytes of memory
BGP attribute entries [18/2952], BGP AS path entries [3/26]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.1         4 64300    1072    1777     5676    0    0 11:08:44 5         
1.1.1.2         4 65000    3252     808     5676    0    0 00:58:55 36        
1.1.1.3         4 65000    3250     807     5676    0    0 00:58:13 36   
</code></pre>
</details>
<details>
<summary>NXOS5</summary>
<pre><code>
NX5# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.5, local AS number 65200
BGP table version is 520, L2VPN EVPN config peers 2, capable peers 2
14 network entries and 18 paths using 3576 bytes of memory
BGP attribute entries [9/1476], BGP AS path entries [2/24]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.2         4 65000    3025    1056      520    0    0 00:59:22 4         
1.1.1.3         4 65000    2994    1051      520    0    0 00:58:42 4      
</code></pre>
</details>
<details>
<summary>NXOS6</summary>
<pre><code>
NX6# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.6, local AS number 65100
BGP table version is 2087, L2VPN EVPN config peers 2, capable peers 2
21 network entries and 27 paths using 5364 bytes of memory
BGP attribute entries [10/1640], BGP AS path entries [2/24]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.2         4 65000    2924    1542     2087    0    0 00:59:49 6         
1.1.1.3         4 65000    2879    1539     2087    0    0 00:59:06 6  
</code></pre>
</details> 
<details>
<summary>NXOS7</summary>
<pre><code>
NX7# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 1.1.1.7, local AS number 65200
BGP table version is 1279, L2VPN EVPN config peers 2, capable peers 2
15 network entries and 19 paths using 3796 bytes of memory
BGP attribute entries [9/1476], BGP AS path entries [2/24]
BGP community entries [0/0], BGP clusterlist entries [0/0]
!
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.2         4 65000    3027    1295     1279    0    0 01:00:10 4         
1.1.1.3         4 65000    2998    1290     1279    0    0 00:59:31 4             
</code></pre>
</details> 

Проверим формирование таблиц маршрутизации:

<details>
  <summary>NXOS1</summary>
<pre><code>
NX1# show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.1, Lo0, [0/0], 18:53:46, direct
1.1.1.1/32, ubest/mbest: 1/0, attached
    *via 1.1.1.1, Lo0, [0/0], 18:53:46, local
1.1.1.2/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/81], 18:52:14, ospf-1, inter
1.1.1.3/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/81], 18:52:14, ospf-1, inter
1.1.1.4/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/41], 18:52:14, ospf-1, intra
1.1.1.5/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/121], 18:52:14, ospf-1, inter
1.1.1.6/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/121], 18:52:14, ospf-1, inter
1.1.1.7/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/121], 18:52:14, ospf-1, inter
10.10.10.0/24, ubest/mbest: 1/0, attached
    *via 10.10.10.252, Vlan10, [0/0], 17:57:47, direct
10.10.10.249/32, ubest/mbest: 1/0, attached
    *via 10.10.10.249, Vlan10, [190/0], 17:26:51, hmm
10.10.10.252/32, ubest/mbest: 1/0, attached
    *via 10.10.10.252, Vlan10, [0/0], 17:57:47, local
10.10.11.0/24, ubest/mbest: 1/0, attached
    *via 10.10.11.252, Vlan11, [0/0], 17:57:03, direct
10.10.11.249/32, ubest/mbest: 1/0, attached
    *via 10.10.11.249, Vlan11, [190/0], 17:26:45, hmm
10.10.11.252/32, ubest/mbest: 1/0, attached
    *via 10.10.11.252, Vlan11, [0/0], 17:57:03, local
10.255.255.255/32, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/121], 18:52:14, ospf-1, inter
172.25.20.0/24, ubest/mbest: 1/0
    *via 1.1.1.4, Eth1/1, [110/80], 18:52:14, ospf-1, intra
!
IP Route Table for VRF "VXLAN_RT"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
192.168.68.219/32, ubest/mbest: 1/0
    *via 10.255.255.255%default, [20/0], 17:52:55, bgp-64300, external, tag 6455
0 (evpn) segid: 9999 tunnelid: 0xaffffff encap: VXLAN
! 
192.168.68.245/32, ubest/mbest: 1/0
    *via 10.255.255.255%default, [20/0], 17:52:55, bgp-64300, external, tag 6455
0 (evpn) segid: 9999 tunnelid: 0xaffffff encap: VXLAN
! 
192.168.69.219/32, ubest/mbest: 1/0
    *via 1.1.1.6%default, [20/0], 17:52:55, bgp-64300, external, tag 64550 (evpn
) segid: 9999 tunnelid: 0x1010106 encap: VXLAN
! 
192.168.69.250/32, ubest/mbest: 1/0
    *via 1.1.1.6%default, [20/0], 17:52:55, bgp-64300, external, tag 64550 (evpn
) segid: 9999 tunnelid: 0x1010106 encap: VXLAN
!
192.168.70.0/24, ubest/mbest: 1/0, attached
    *via 192.168.70.252, Vlan70, [0/0], 17:53:39, direct
192.168.70.219/32, ubest/mbest: 1/0, attached
    *via 192.168.70.219, Vlan70, [190/0], 17:23:14, hmm
192.168.70.249/32, ubest/mbest: 1/0, attached
    *via 192.168.70.249, Vlan70, [190/0], 17:26:40, hmm
192.168.70.252/32, ubest/mbest: 1/0, attached
    *via 192.168.70.252, Vlan70, [0/0], 17:53:39, local
</code></pre>
</details> 
<details>
  <summary>NXOS2</summary>
<pre><code>
NX2#  show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.2, Lo0, [0/0], 3d20h, direct
1.1.1.1/32, ubest/mbest: 1/0
    *via 172.25.20.3, Eth1/4, [110/81], 18:53:31, ospf-1, intra
1.1.1.2/32, ubest/mbest: 1/0, attached
    *via 1.1.1.2, Lo0, [0/0], 3d20h, local
1.1.1.3/32, ubest/mbest: 3/0
    *via 1.1.1.5, Eth1/3, [110/81], 2d16h, ospf-1, intra
    *via 1.1.1.6, Eth1/1, [110/81], 2d16h, ospf-1, intra
    *via 1.1.1.7, Eth1/2, [110/81], 2d16h, ospf-1, intra
1.1.1.4/32, ubest/mbest: 1/0
    *via 172.25.20.3, Eth1/4, [110/41], 19:01:31, ospf-1, intra
1.1.1.5/32, ubest/mbest: 1/0
    *via 1.1.1.5, Eth1/3, [110/41], 2d17h, ospf-1, intra
1.1.1.6/32, ubest/mbest: 1/0
    *via 1.1.1.6, Eth1/1, [110/41], 3d20h, ospf-1, intra
1.1.1.7/32, ubest/mbest: 1/0
    *via 1.1.1.7, Eth1/2, [110/41], 2d17h, ospf-1, intra
10.255.255.255/32, ubest/mbest: 2/0
    *via 1.1.1.5, Eth1/3, [110/41], 2d17h, ospf-1, intra
    *via 1.1.1.7, Eth1/2, [110/41], 2d17h, ospf-1, intra
172.25.20.0/24, ubest/mbest: 1/0, attached
    *via 172.25.20.1, Eth1/4, [0/0], 19:04:12, direct
172.25.20.1/32, ubest/mbest: 1/0, attached
    *via 172.25.20.1, Eth1/4, [0/0], 19:04:12, local
</code></pre>
</details>
<details>
  <summary>NXOS3</summary>
<pre><code>
NX3# show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.3, Lo0, [0/0], 3d20h, direct
1.1.1.1/32, ubest/mbest: 1/0
    *via 172.25.20.3, Eth1/4, [110/81], 18:54:26, ospf-1, intra
1.1.1.2/32, ubest/mbest: 3/0
    *via 1.1.1.5, Eth1/3, [110/81], 2d16h, ospf-1, intra
    *via 1.1.1.6, Eth1/1, [110/81], 2d16h, ospf-1, intra
    *via 1.1.1.7, Eth1/2, [110/81], 2d16h, ospf-1, intra
1.1.1.3/32, ubest/mbest: 1/0, attached
    *via 1.1.1.3, Lo0, [0/0], 3d20h, local
1.1.1.4/32, ubest/mbest: 1/0
    *via 172.25.20.3, Eth1/4, [110/41], 19:03:06, ospf-1, intra
1.1.1.5/32, ubest/mbest: 1/0
    *via 1.1.1.5, Eth1/3, [110/41], 2d16h, ospf-1, intra
1.1.1.6/32, ubest/mbest: 1/0
    *via 1.1.1.6, Eth1/1, [110/41], 2d16h, ospf-1, intra
1.1.1.7/32, ubest/mbest: 1/0
    *via 1.1.1.7, Eth1/2, [110/41], 2d16h, ospf-1, intra
10.255.255.255/32, ubest/mbest: 2/0
    *via 1.1.1.5, Eth1/3, [110/41], 2d16h, ospf-1, intra
    *via 1.1.1.7, Eth1/2, [110/41], 2d16h, ospf-1, intra
172.25.20.0/24, ubest/mbest: 1/0, attached
    *via 172.25.20.2, Eth1/4, [0/0], 19:03:55, direct
172.25.20.2/32, ubest/mbest: 1/0, attached
    *via 172.25.20.2, Eth1/4, [0/0], 19:03:55, local
</code></pre>
</details>
<details>
  <summary>NXOS4</summary>
<pre><code>
NX4# show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.4, Lo0, [0/0], 19:17:08, direct
1.1.1.1/32, ubest/mbest: 1/0
    *via 1.1.1.1, Eth1/1, [110/41], 18:55:25, ospf-1, intra
1.1.1.2/32, ubest/mbest: 1/0
    *via 172.25.20.1, Eth1/2, [110/41], 19:03:20, ospf-1, inter
1.1.1.3/32, ubest/mbest: 1/0
    *via 172.25.20.2, Eth1/2, [110/41], 19:04:06, ospf-1, inter
1.1.1.4/32, ubest/mbest: 1/0, attached
    *via 1.1.1.4, Lo0, [0/0], 19:17:08, local
1.1.1.5/32, ubest/mbest: 2/0
    *via 172.25.20.1, Eth1/2, [110/81], 19:03:20, ospf-1, inter
    *via 172.25.20.2, Eth1/2, [110/81], 19:03:20, ospf-1, inter
1.1.1.6/32, ubest/mbest: 2/0
    *via 172.25.20.1, Eth1/2, [110/81], 19:03:20, ospf-1, inter
    *via 172.25.20.2, Eth1/2, [110/81], 19:03:20, ospf-1, inter
1.1.1.7/32, ubest/mbest: 2/0
    *via 172.25.20.1, Eth1/2, [110/81], 19:03:20, ospf-1, inter
    *via 172.25.20.2, Eth1/2, [110/81], 19:03:20, ospf-1, inter
10.255.255.255/32, ubest/mbest: 2/0
    *via 172.25.20.1, Eth1/2, [110/81], 19:03:20, ospf-1, inter
    *via 172.25.20.2, Eth1/2, [110/81], 19:03:20, ospf-1, inter
172.25.20.0/24, ubest/mbest: 1/0, attached
    *via 172.25.20.3, Eth1/2, [0/0], 19:08:07, direct
172.25.20.3/32, ubest/mbest: 1/0, attached
    *via 172.25.20.3, Eth1/2, [0/0], 19:08:07, local
</code></pre>
</details>
<details>
  <summary>NXOS5</summary>
<pre><code>
NX5# show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.5, Lo0, [0/0], 2d17h, direct
1.1.1.1/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/121], 18:56:03, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/121], 18:56:03, ospf-1, inter
1.1.1.2/32, ubest/mbest: 1/0
    *via 1.1.1.2, Eth1/1, [110/41], 2d17h, ospf-1, intra
1.1.1.3/32, ubest/mbest: 1/0
    *via 1.1.1.3, Eth1/2, [110/41], 2d17h, ospf-1, intra
1.1.1.4/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 19:04:03, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/81], 19:04:03, ospf-1, inter
1.1.1.5/32, ubest/mbest: 1/0, attached
    *via 1.1.1.5, Lo0, [0/0], 2d17h, local
1.1.1.6/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
1.1.1.7/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
10.10.10.0/24, ubest/mbest: 1/0, attached
    *via 10.10.10.253, Vlan10, [0/0], 2d17h, direct
10.10.10.245/32, ubest/mbest: 1/0, attached
    *via 10.10.10.245, Vlan10, [190/0], 2d15h, hmm
10.10.10.253/32, ubest/mbest: 1/0, attached
    *via 10.10.10.253, Vlan10, [0/0], 2d17h, local
10.10.11.0/24, ubest/mbest: 1/0, attached
    *via 10.10.11.253, Vlan11, [0/0], 2d17h, direct
10.10.11.245/32, ubest/mbest: 1/0, attached
    *via 10.10.11.245, Vlan11, [190/0], 2d15h, hmm
10.10.11.253/32, ubest/mbest: 1/0, attached
    *via 10.10.11.253, Vlan11, [0/0], 2d17h, local
10.255.255.255/32, ubest/mbest: 2/0, attached
    *via 10.255.255.255, Lo0, [0/0], 2d17h, local
    *via 10.255.255.255, Lo0, [0/0], 2d17h, direct
172.25.20.0/24, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/80], 19:04:43, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/80], 19:04:43, ospf-1, inter
!
IP Route Table for VRF "VPC"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
10.15.2.0/31, ubest/mbest: 1/0, attached
    *via 10.15.2.1, Eth1/3, [0/0], 2d17h, direct
10.15.2.1/32, ubest/mbest: 1/0, attached
    *via 10.15.2.1, Eth1/3, [0/0], 2d17h, local
!
IP Route Table for VRF "VXLAN_RT"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
192.168.68.0/24, ubest/mbest: 1/0, attached
    *via 192.168.68.253, Vlan68, [0/0], 2d13h, direct
192.168.68.219/32, ubest/mbest: 1/0, attached
    *via 192.168.68.219, Vlan68, [190/0], 2d13h, hmm
192.168.68.245/32, ubest/mbest: 1/0, attached
    *via 192.168.68.245, Vlan68, [190/0], 2d13h, hmm
192.168.68.253/32, ubest/mbest: 1/0, attached
    *via 192.168.68.253, Vlan68, [0/0], 2d13h, local
192.168.69.219/32, ubest/mbest: 1/0
    *via 1.1.1.6%default, [20/0], 07:09:09, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010106 encap: VXLAN
! 
192.168.69.250/32, ubest/mbest: 1/0
    *via 1.1.1.6%default, [20/0], 07:09:09, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010106 encap: VXLAN
! 
192.168.70.219/32, ubest/mbest: 1/0
    *via 1.1.1.1%default, [20/0], 17:27:02, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010101 encap: VXLAN
!
192.168.70.249/32, ubest/mbest: 1/0
    *via 1.1.1.1%default, [20/0], 17:30:28, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010101 encap: VXLAN
</code></pre>
</details>
<details>
  <summary>NXOS6</summary>
<pre><code>
NX6# show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.6, Lo0, [0/0], 3d20h, direct
1.1.1.1/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/121], 18:57:10, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/121], 18:57:10, ospf-1, inter
1.1.1.2/32, ubest/mbest: 1/0
    *via 1.1.1.2, Eth1/1, [110/41], 3d20h, ospf-1, intra
1.1.1.3/32, ubest/mbest: 1/0
    *via 1.1.1.3, Eth1/2, [110/41], 2d17h, ospf-1, intra
1.1.1.4/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 19:05:10, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/81], 19:05:10, ospf-1, inter
1.1.1.5/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
1.1.1.6/32, ubest/mbest: 1/0, attached
    *via 1.1.1.6, Lo0, [0/0], 3d20h, local
1.1.1.7/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
10.10.10.0/24, ubest/mbest: 1/0, attached
    *via 10.10.10.251, Vlan10, [0/0], 2d16h, direct
10.10.10.1/32, ubest/mbest: 1/0, attached
    *via 10.10.10.1, Vlan10, [190/0], 2d16h, hmm
10.10.10.250/32, ubest/mbest: 1/0, attached
    *via 10.10.10.250, Vlan10, [190/0], 2d16h, hmm
10.10.10.251/32, ubest/mbest: 1/0, attached
    *via 10.10.10.251, Vlan10, [0/0], 2d16h, local
10.10.11.0/24, ubest/mbest: 1/0, attached
    *via 10.10.11.251, Vlan11, [0/0], 2d16h, direct
10.10.11.1/32, ubest/mbest: 1/0, attached
    *via 10.10.11.1, Vlan11, [190/0], 2d16h, hmm
10.10.11.250/32, ubest/mbest: 1/0, attached
    *via 10.10.11.250, Vlan11, [190/0], 2d16h, hmm
10.10.11.251/32, ubest/mbest: 1/0, attached
    *via 10.10.11.251, Vlan11, [0/0], 2d16h, local
10.255.255.255/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
172.25.20.0/24, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/80], 19:05:50, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/80], 19:05:50, ospf-1, inter
!
IP Route Table for VRF "VXLAN_RT"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
192.168.68.219/32, ubest/mbest: 1/0
    *via 10.255.255.255%default, [20/0], 20:00:52, bgp-65100, external, tag 6500
0 (evpn) segid: 9999 tunnelid: 0xaffffff encap: VXLAN
! 
192.168.68.245/32, ubest/mbest: 1/0
    *via 10.255.255.255%default, [20/0], 20:00:52, bgp-65100, external, tag 6500
0 (evpn) segid: 9999 tunnelid: 0xaffffff encap: VXLAN
! 
192.168.69.0/24, ubest/mbest: 1/0, attached
    *via 192.168.69.251, Vlan69, [0/0], 2d13h, direct
192.168.69.219/32, ubest/mbest: 1/0, attached
    *via 192.168.69.219, Vlan69, [190/0], 2d13h, hmm
192.168.69.250/32, ubest/mbest: 1/0, attached
    *via 192.168.69.250, Vlan69, [190/0], 2d09h, hmm
192.168.69.251/32, ubest/mbest: 1/0, attached
    *via 192.168.69.251, Vlan69, [0/0], 2d13h, local
192.168.70.219/32, ubest/mbest: 1/0
    *via 1.1.1.1%default, [20/0], 17:28:09, bgp-65100, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010101 encap: VXLAN
!
192.168.70.249/32, ubest/mbest: 1/0
    *via 1.1.1.1%default, [20/0], 17:31:34, bgp-65100, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010101 encap: VXLAN
</code></pre>
</details>
<details>
  <summary>NXOS7</summary>
<pre><code>
NX7# show ip route vrf all 
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
1.1.1.0/24, ubest/mbest: 1/0, attached
    *via 1.1.1.7, Lo0, [0/0], 2d17h, direct
1.1.1.1/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/121], 18:58:34, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/121], 18:58:34, ospf-1, inter
1.1.1.2/32, ubest/mbest: 1/0
    *via 1.1.1.2, Eth1/1, [110/41], 2d17h, ospf-1, intra
1.1.1.3/32, ubest/mbest: 1/0
    *via 1.1.1.3, Eth1/2, [110/41], 2d17h, ospf-1, intra
1.1.1.4/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 19:06:34, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/81], 19:06:34, ospf-1, inter
1.1.1.5/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
1.1.1.6/32, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/81], 2d17h, ospf-1, intra
    *via 1.1.1.3, Eth1/2, [110/81], 2d17h, ospf-1, intra
1.1.1.7/32, ubest/mbest: 1/0, attached
    *via 1.1.1.7, Lo0, [0/0], 2d17h, local
10.10.10.0/24, ubest/mbest: 1/0, attached
    *via 10.10.10.253, Vlan10, [0/0], 2d17h, direct
10.10.10.245/32, ubest/mbest: 1/0, attached
    *via 10.10.10.245, Vlan10, [190/0], 2d15h, hmm
10.10.10.253/32, ubest/mbest: 1/0, attached
    *via 10.10.10.253, Vlan10, [0/0], 2d17h, local
10.10.11.0/24, ubest/mbest: 1/0, attached
    *via 10.10.11.253, Vlan11, [0/0], 2d17h, direct
10.10.11.245/32, ubest/mbest: 1/0, attached
    *via 10.10.11.245, Vlan11, [190/0], 2d15h, hmm
10.10.11.253/32, ubest/mbest: 1/0, attached
    *via 10.10.11.253, Vlan11, [0/0], 2d17h, local
10.255.255.255/32, ubest/mbest: 2/0, attached
    *via 10.255.255.255, Lo0, [0/0], 2d17h, local
    *via 10.255.255.255, Lo0, [0/0], 2d17h, direct
172.25.20.0/24, ubest/mbest: 2/0
    *via 1.1.1.2, Eth1/1, [110/80], 19:07:14, ospf-1, inter
    *via 1.1.1.3, Eth1/2, [110/80], 19:07:14, ospf-1, inter
!
IP Route Table for VRF "VPC"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
10.15.2.0/31, ubest/mbest: 1/0, attached
    *via 10.15.2.0, Eth1/3, [0/0], 2d17h, direct
10.15.2.0/32, ubest/mbest: 1/0, attached
    *via 10.15.2.0, Eth1/3, [0/0], 2d17h, local
!
IP Route Table for VRF "VXLAN_RT"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>
!
192.168.68.0/24, ubest/mbest: 1/0, attached
    *via 192.168.68.253, Vlan68, [0/0], 2d13h, direct
192.168.68.219/32, ubest/mbest: 1/0, attached
    *via 192.168.68.219, Vlan68, [190/0], 2d13h, hmm
192.168.68.245/32, ubest/mbest: 1/0, attached
    *via 192.168.68.245, Vlan68, [190/0], 2d13h, hmm
192.168.68.253/32, ubest/mbest: 1/0, attached
    *via 192.168.68.253, Vlan68, [0/0], 2d13h, local
192.168.69.219/32, ubest/mbest: 1/0
    *via 1.1.1.6%default, [20/0], 07:11:40, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010106 encap: VXLAN
! 
192.168.69.250/32, ubest/mbest: 1/0
    *via 1.1.1.6%default, [20/0], 07:11:40, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010106 encap: VXLAN
! 
192.168.70.219/32, ubest/mbest: 1/0
    *via 1.1.1.1%default, [20/0], 17:29:33, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010101 encap: VXLAN
!
192.168.70.249/32, ubest/mbest: 1/0
    *via 1.1.1.1%default, [20/0], 17:32:58, bgp-65200, external, tag 65000 (evpn
) segid: 9999 tunnelid: 0x1010101 encap: VXLAN
</code></pre>
</details>

Теперь проверим nve peers и таблицу для BGP EVPN:

<details>
  <summary>NXOS1</summary>
<pre><code>
NX1# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      1.1.1.6          Up    CP        18:01:03 5000.0006.0007   
nve1      10.255.255.255   Up    CP        18:01:03 5000.0007.0007  
!
!
NX1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 863, Local Router ID is 1.1.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2
!
   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1.1.1.1:32837    (L2VNI 10070)
*>l[2]:[0]:[0]:[48]:[0050.7966.6815]:[0]:[0.0.0.0]/216
                      1.1.1.1                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.b000]:[0]:[0.0.0.0]/216
                      1.1.1.1                           100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.7966.6815]:[32]:[192.168.70.219]/272
                      1.1.1.1                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.b000]:[32]:[192.168.70.249]/272
                      1.1.1.1                           100      32768 i
*>l[3]:[0]:[32]:[1.1.1.1]/88
                      1.1.1.1                           100      32768 i
!
Route Distinguisher: 1.1.1.5:32835
*>e[2]:[0]:[0]:[48]:[0050.7966.6813]:[32]:[192.168.68.219]/272
                      10.255.255.255                                 0 64550 650
00 65200 i
*>e[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[192.168.68.245]/272
                      10.255.255.255                                 0 64550 650
00 65200 i
!
Route Distinguisher: 1.1.1.6:32836
*>e[2]:[0]:[0]:[48]:[0050.7966.6814]:[32]:[192.168.69.219]/272
                      1.1.1.6                                        0 64550 650
00 65100 i
*>e[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[32]:[192.168.69.250]/272
                      1.1.1.6                                        0 64550 650
00 65100 i
!
Route Distinguisher: 1.1.1.7:32835
*>e[2]:[0]:[0]:[48]:[0050.7966.6813]:[32]:[192.168.68.219]/272
                      10.255.255.255                                 0 64550 650
00 65200 i
*>e[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[192.168.68.245]/272
                      10.255.255.255                                 0 64550 650
00 65200 i
</code></pre>
</details>
<details>
  <summary>NXOS5</summary>
<pre><code>
NX5# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      1.1.1.1          Up    CP        17:35:56 5000.0001.0007   
nve1      1.1.1.6          Up    CP        20:32:50 5000.0006.0007   
!
!
NX5# show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 752, Local Router ID is 1.1.1.5
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2
!
   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1.1.1.1:32837
* e[2]:[0]:[0]:[48]:[0050.7966.6815]:[32]:[192.168.70.219]/272
                      1.1.1.1                                        0 65000 645
50 64300 i
*>e                   1.1.1.1                                        0 65000 645
50 64300 i
* e[2]:[0]:[0]:[48]:[aabb.cc80.b000]:[32]:[192.168.70.249]/272
                      1.1.1.1                                        0 65000 645
50 64300 i
*>e                   1.1.1.1                                        0 65000 645
50 64300 i
!
Route Distinguisher: 1.1.1.5:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[10.10.10.245]/248
                      10.255.255.255                    100      32768 i
*>l[3]:[0]:[32]:[10.255.255.255]/88
                      10.255.255.255                    100      32768 i
!
Route Distinguisher: 1.1.1.5:32778    (L2VNI 10011)
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[10.10.11.245]/248
                      10.255.255.255                    100      32768 i
*>l[3]:[0]:[32]:[10.255.255.255]/88
                      10.255.255.255                    100      32768 i
!
Route Distinguisher: 1.1.1.5:32835    (L2VNI 10068)
*>l[2]:[0]:[0]:[48]:[0050.7966.6813]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.7966.6813]:[32]:[192.168.68.219]/272
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[192.168.68.245]/272
                      10.255.255.255                    100      32768 i
*>l[3]:[0]:[32]:[10.255.255.255]/88
                      10.255.255.255                    100      32768 i
!
Route Distinguisher: 1.1.1.6:32836
*>e[2]:[0]:[0]:[48]:[0050.7966.6814]:[32]:[192.168.69.219]/272
                      1.1.1.6                                        0 65000 651
00 i
* e                   1.1.1.6                                        0 65000 651
00 i
*>e[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[32]:[192.168.69.250]/272
                      1.1.1.6                                        0 65000 651
00 i
* e                   1.1.1.6                                        0 65000 651
00 i
</code></pre>
</details>
<details>
  <summary>NXOS6</summary>
<pre><code>
NX6# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      1.1.1.1          Up    CP        17:37:28 5000.0001.0007   
nve1      10.255.255.255   Up    CP        20:34:22 5000.0005.0007   
!
!
NX6(config)# show bgp l2vpn evpn 
NX6#  show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 2979, Local Router ID is 1.1.1.6
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2
!
   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1.1.1.1:32837
* e[2]:[0]:[0]:[48]:[0050.7966.6815]:[32]:[192.168.70.219]/272
                      1.1.1.1                                        0 65000 645
50 64300 i
*>e                   1.1.1.1                                        0 65000 645
50 64300 i
* e[2]:[0]:[0]:[48]:[aabb.cc80.b000]:[32]:[192.168.70.249]/272
                      1.1.1.1                                        0 65000 645
50 64300 i
*>e                   1.1.1.1                                        0 65000 645
50 64300 i
!
Route Distinguisher: 1.1.1.5:32835
*>e[2]:[0]:[0]:[48]:[0050.7966.6813]:[32]:[192.168.68.219]/272
                      10.255.255.255                                 0 65000 652
00 i
* e                   10.255.255.255                                 0 65000 652
00 i
*>e[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[192.168.68.245]/272
                      10.255.255.255                                 0 65000 652
00 i
* e                   10.255.255.255                                 0 65000 652
00 i
!
Route Distinguisher: 1.1.1.6:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[0050.7966.680c]:[0]:[0.0.0.0]/216
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[0]:[0.0.0.0]/216
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.7966.680c]:[32]:[10.10.10.1]/248
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[32]:[10.10.10.250]/248
                      1.1.1.6                           100      32768 i
*>l[3]:[0]:[32]:[1.1.1.6]/88
                      1.1.1.6                           100      32768 i
!
Route Distinguisher: 1.1.1.6:32778    (L2VNI 10011)
*>l[2]:[0]:[0]:[48]:[0050.7966.6811]:[0]:[0.0.0.0]/216
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[0]:[0.0.0.0]/216
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.7966.6811]:[32]:[10.10.11.1]/248
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[32]:[10.10.11.250]/248
                      1.1.1.6                           100      32768 i
*>l[3]:[0]:[32]:[1.1.1.6]/88
                      1.1.1.6                           100      32768 i
!
Route Distinguisher: 1.1.1.6:32836    (L2VNI 10069)
*>l[2]:[0]:[0]:[48]:[0050.7966.6814]:[0]:[0.0.0.0]/216
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[0]:[0.0.0.0]/216
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.7966.6814]:[32]:[192.168.69.219]/272
                      1.1.1.6                           100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[32]:[192.168.69.250]/272
                      1.1.1.6                           100      32768 i
*>l[3]:[0]:[32]:[1.1.1.6]/88
                      1.1.1.6                           100      32768 i
!
Route Distinguisher: 1.1.1.7:32835
*>e[2]:[0]:[0]:[48]:[0050.7966.6813]:[32]:[192.168.68.219]/272
                      10.255.255.255                                 0 65000 652
00 i
* e                   10.255.255.255                                 0 65000 652
00 i
*>e[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[192.168.68.245]/272
                      10.255.255.255                                 0 65000 652
00 i
* e                   10.255.255.255                                 0 65000 652
00 i
</code></pre>
</details>
<details>
  <summary>NXOS7</summary>
<pre><code>
NX7(config)# show nve peers 
Interface Peer-IP          State LearnType Uptime   Router-Mac       
--------- ---------------  ----- --------- -------- -----------------
nve1      1.1.1.1          Up    CP        17:39:20 5000.0001.0007   
nve1      1.1.1.6          Up    CP        20:36:14 5000.0006.0007   
! 
!
NX7(config)# show bgp l2vpn evpn 
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 1823, Local Router ID is 1.1.1.7
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2
!
   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 1.1.1.1:32837
* e[2]:[0]:[0]:[48]:[0050.7966.6815]:[32]:[192.168.70.219]/272
                      1.1.1.1                                        0 65000 645
50 64300 i
*>e                   1.1.1.1                                        0 65000 645
50 64300 i
* e[2]:[0]:[0]:[48]:[aabb.cc80.b000]:[32]:[192.168.70.249]/272
                      1.1.1.1                                        0 65000 645
50 64300 i
*>e                   1.1.1.1                                        0 65000 645
50 64300 i
!
Route Distinguisher: 1.1.1.6:32836
* e[2]:[0]:[0]:[48]:[0050.7966.6814]:[32]:[192.168.69.219]/272
                      1.1.1.6                                        0 65000 651
00 i
*>e                   1.1.1.6                                        0 65000 651
00 i
* e[2]:[0]:[0]:[48]:[aabb.cc80.9000]:[32]:[192.168.69.250]/272
                      1.1.1.6                                        0 65000 651
00 i
*>e                   1.1.1.6                                        0 65000 651
00 i
!
Route Distinguisher: 1.1.1.7:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[10.10.10.245]/248
                      10.255.255.255                    100      32768 i
*>l[3]:[0]:[32]:[10.255.255.255]/88
                      10.255.255.255                    100      32768 i
!
Route Distinguisher: 1.1.1.7:32778    (L2VNI 10011)
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[10.10.11.245]/248
                      10.255.255.255                    100      32768 i
*>l[3]:[0]:[32]:[10.255.255.255]/88
                      10.255.255.255                    100      32768 i
!
Route Distinguisher: 1.1.1.7:32835    (L2VNI 10068)
*>l[2]:[0]:[0]:[48]:[0050.7966.6813]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[0]:[0.0.0.0]/216
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.7966.6813]:[32]:[192.168.68.219]/272
                      10.255.255.255                    100      32768 i
*>l[2]:[0]:[0]:[48]:[aabb.cc80.a000]:[32]:[192.168.68.245]/272
                      10.255.255.255                    100      32768 i
*>l[3]:[0]:[32]:[10.255.255.255]/88
                      10.255.255.255                    100      32768 i
</code></pre>
</details>

Проверка связности через утилиту ping

<details>
  <summary>VPC</summary>
<pre><code>
VPCS> show ip
!
NAME        : VPCS[1]
IP/MASK     : 192.168.70.219/24
GATEWAY     : 192.168.70.252
DNS         : 
MAC         : 00:50:79:66:68:15
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
!
VPCS> ping 192.168.68.219
!
84 bytes from 192.168.68.219 icmp_seq=1 ttl=62 time=46.546 ms
84 bytes from 192.168.68.219 icmp_seq=2 ttl=62 time=37.931 ms
84 bytes from 192.168.68.219 icmp_seq=3 ttl=62 time=45.442 ms
84 bytes from 192.168.68.219 icmp_seq=4 ttl=62 time=30.613 ms
84 bytes from 192.168.68.219 icmp_seq=5 ttl=62 time=42.804 ms
!
VPCS> ping 192.168.69.219
!
84 bytes from 192.168.69.219 icmp_seq=1 ttl=62 time=33.773 ms
84 bytes from 192.168.69.219 icmp_seq=2 ttl=62 time=28.281 ms
84 bytes from 192.168.69.219 icmp_seq=3 ttl=62 time=32.808 ms
84 bytes from 192.168.69.219 icmp_seq=4 ttl=62 time=33.742 ms
84 bytes from 192.168.69.219 icmp_seq=5 ttl=62 time=46.666 ms
</code></pre>
</details>

Вывод:

- Настроил BGP peering между Spine в одной зоне и во второй
- Все клиенты имеют L2/L3 связанность

Multicast на данный момент считается уже устаревшим решением, поэтому главной задачей данного проекта перевести сеть на EVPN

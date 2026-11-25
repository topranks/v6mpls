# sr-mpls-v6-xrd: IPv4 on the PE core interfaces

Testing whether enabling IPv4 on the PEs' P-facing interfaces lets IPv4 cross the IPv6 SR-MPLS LSPs.  On Junos a bare `family inet` on the core interfaces fixed implicit null and non-null UHP (see [../sr-mpls-v6-junos/BGP_LU_NOTES.md](../sr-mpls-v6-junos/BGP_LU_NOTES.md#addendum-family-inet-on-core-interfaces)).  This is the XR equivalent: an IPv4 loopback on each PE and `ipv4 unnumbered` on the core interface.  P has no IPv4 configured.  Image is XRd control plane 24.2.11.

## Summary

**IPv4 still fails on XRd in all three PHP modes.**  With IPv4 enabled on the core interface, PE1 now does transmit the IPv4 packets towards P (before, it dropped them).  But it sends them with the MPLS Ethertype (0x8847) and **no label**: the IPv4 header follows the Ethernet header directly.  P parses the first 4 bytes of the IPv4 header as a label and drops the packet.

PE1's CEF entry for the IPv4 route recurses via PE2's IPv6 loopback but has no label imposition, while the IPv6 entry for that loopback has `labels imposed {16003}`.  Re-installing the route (BGP neighbor shut / no shut) didn't change this.

| Mode | P outgoing label for 16003 | Frame PE1 → P | IPv4 ping |
|---|---|---|---|
| UHP / Explicit Null | Exp-Null-v6 (2) | 0x8847, no label, IPv4 header | Fail |
| PHP / Implicit Null | Pop | 0x8847, no label, IPv4 header | Fail |
| UHP / non-null label | 16003 | 0x8847, no label, IPv4 header | Fail |

IPv6 works throughout.

## Config

PE1 (PE2 is the same with 192.0.2.3).  The loopback addresses are from 192.0.2.0/24 (TEST-NET-1) and aren't advertised anywhere; IS-IS only has the IPv6 address family.
```
interface Loopback0
 ipv4 address 192.0.2.1 255.255.255.255
interface GigabitEthernet0/0/0/1
 ipv4 point-to-point
 ipv4 unnumbered Loopback0
```
`ipv4 unnumbered` alone is rejected on an Ethernet interface; `ipv4 point-to-point` is needed first:
```
!! SEMANTIC ERRORS: This configuration was rejected by 
!! the system due to semantic errors. The individual 
!! errors with each failed configuration command can be 
!! found below.


interface GigabitEthernet0/0/0/1
 ipv4 unnumbered Loopback0
!!% 'ipv4-ma' detected the 'warning' condition 'Point-to-point (non-multi-access) interfaces only'
!
end
```
```
RP/0/RP0/CPU0:pe1#show ipv4 interface brief

Interface                      IP-Address      Status          Protocol Vrf-Name
Loopback0                      192.0.2.1       Up              Up       default 
MgmtEth0/RP0/CPU0/0            172.20.20.5     Up              Up       default 
GigabitEthernet0/0/0/0         198.51.100.1    Up              Up       default 
GigabitEthernet0/0/0/1         192.0.2.1       Up              Up       default 
```

## Results (explicit null, the startup config)

```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4104ms
```
PE1's interface accounting now shows IPv4 packets going out of the core interface, 84 bytes each (one per echo request).  The single ARP is a gratuitous ARP for 192.0.2.1 sent when IPv4 came up on the interface (see the capture below); PE1 doesn't ARP for anything when forwarding the pings:
```
RP/0/RP0/CPU0:pe1#show interfaces GigabitEthernet0/0/0/1 accounting
GigabitEthernet0/0/0/1
  Protocol              Pkts In         Chars In     Pkts Out        Chars Out
  IPV4_UNICAST                0                0            5              420
  IPV6_UNICAST                4              344            2              144
  IPV6_MULTICAST             11             1058            0                0
  MPLS                       16             1722           14             1483
  ARP                         0                0            1               42
  IPV6_ND                    13             1230           15             1192
  CLNS                       42            50988           41            50229
```
```
RP/0/RP0/CPU0:pe1#show arp GigabitEthernet0/0/0/1

-------------------------------------------------------------------------------
0/RP0/CPU0
-------------------------------------------------------------------------------
Address         Age        Hardware Addr   State      Type  Interface
192.0.2.1       -          aac1.abaa.3c80  Interface  ARPA  GigabitEthernet0/0/0/1
```
Captured in P's netns on Gi0-0-0-0 (facing PE1), [unnum_srxrd_expnull_pe1_to_p.pcap](pcaps/unnum_srxrd_expnull_pe1_to_p.pcap).  Frame 6 is the gratuitous ARP, triggered by removing and re-adding `ipv4 unnumbered` during the capture.  tshark decodes the IPv4 header as a bogus label stack:
```
1	0.000000000	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	282624,375492,258070	MPLS Label Switched Packet
2	1.029198000	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	282624,378436,258070	MPLS Label Switched Packet
3	2.053230000	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	282624,381972,258070,831495	MPLS Label Switched Packet
4	3.077083000	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	282624,385108,258070	MPLS Label Switched Packet
5	4.101164000	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	282624,385860,258070	MPLS Label Switched Packet
6	32.681841000	aa:c1:ab:aa:3c:80	ff:ff:ff:ff:ff:ff	0x0806		Gratuitous ARP for 192.0.2.1 (Reply)
```
The hex shows Ethertype `88 47` followed immediately by `45 00 00 54`, the start of the IPv4 header (203.0.113.1 → 203.0.113.2 is `cb 00 71 01` → `cb 00 71 02`).  282624 is 0x45000, i.e. those first bytes read as a label:
```
0000  aa c1 ab 93 75 3c aa c1 ab aa 3c 80 88 47 45 00   ....u<....<..GE.
0010  00 54 5b ac 40 00 3f 01 67 f8 cb 00 71 01 cb 00   .T[.@.?.g...q...
0020  71 02 08 00 4f b0 00 39 00 01 08 22 06 6b 00 00   q...O..9...".k..
0030  00 00 da b5 00 00 00 00 00 00 10 11 12 13 14 15   ................
0040  16 17 18 19 1a 1b 1c 1d 1e 1f 20 21 22 23 24 25   .......... !"#$%
0050  26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35   &'()*+,-./012345
0060  36 37                                             67
```
```
Address Resolution Protocol (reply/gratuitous ARP)
    Hardware type: Ethernet (1)
    Protocol type: IPv4 (0x0800)
    Hardware size: 6
    Protocol size: 4
    Opcode: reply (2)
    [Is gratuitous: True]
    Sender MAC address: aa:c1:ab:aa:3c:80
    Sender IP address: 192.0.2.1
    Target MAC address: ff:ff:ff:ff:ff:ff
    Target IP address: 192.0.2.1
```

## Implicit null and non-null UHP

Changed on both PEs (index 1 on PE1, 3 on PE2):
```
router isis IGP
 interface Loopback0
  address-family ipv6 unicast
   no prefix-sid index 1 explicit-null
   prefix-sid index 1
```
```
RP/0/RP0/CPU0:p#show mpls forwarding labels 16003
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  Pop         SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:fe50:8eb1   \
                                                                   0           
```
```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4073ms
```
[unnum_srxrd_implicit_null_pe1_to_p.pcap](pcaps/unnum_srxrd_implicit_null_pe1_to_p.pcap):
```
1	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	MPLS Label Switched Packet
2	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	MPLS Label Switched Packet
3	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	MPLS Label Switched Packet
4	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	MPLS Label Switched Packet
5	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	0x8847	MPLS Label Switched Packet
```
```
0000  aa c1 ab 93 75 3c aa c1 ab aa 3c 80 88 47 45 00   ....u<....<..GE.
0010  00 54 74 5d 40 00 3f 01 4f 47 cb 00 71 01 cb 00   .Tt]@.?.OG..q...
0020  71 02 08 00 8f 8a 00 3a 00 01 15 23 06 6b 00 00   q......:...#.k..
0030  00 00 86 d9 07 00 00 00 00 00 10 11 12 13 14 15   ................
0040  16 17 18 19 1a 1b 1c 1d 1e 1f 20 21 22 23 24 25   .......... !"#$%
0050  26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35   &'()*+,-./012345
0060  36 37                                             67
```
Then `prefix-sid index N php-disable` on both PEs:
```
RP/0/RP0/CPU0:p#show mpls forwarding labels 16003
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  16003       SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:fe50:8eb1   \
                                                                   0           
```
```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4105ms
```
[unnum_srxrd_uhp_nonnull_pe1_to_p.pcap](pcaps/unnum_srxrd_uhp_nonnull_pe1_to_p.pcap), same again:
```
0000  aa c1 ab 93 75 3c aa c1 ab aa 3c 80 88 47 45 00   ....u<....<..GE.
0010  00 54 a8 e7 40 00 3f 01 1a bd cb 00 71 01 cb 00   .T..@.?.....q...
0020  71 02 08 00 ee ea 00 3b 00 01 b1 23 06 6b 00 00   q......;...#.k..
0030  00 00 8d 77 05 00 00 00 00 00 10 11 12 13 14 15   ...w............
0040  16 17 18 19 1a 1b 1c 1d 1e 1f 20 21 22 23 24 25   .......... !"#$%
0050  26 27 28 29 2a 2b 2c 2d 2e 2f 30 31 32 33 34 35   &'()*+,-./012345
0060  36 37                                             67
```

## CEF on PE1

After shutting and re-enabling the iBGP neighbor (to rule out a stale entry from before IPv4 was enabled), the IPv4 route has no label imposition:
```
RP/0/RP0/CPU0:pe1#show cef ipv4 203.0.113.2/32 detail
203.0.113.2/32, version 11, internal 0x5000001 0x40 (ptr 0x87203ea0) [1], 0x0 (0x0), 0x0 (0x0)
 Updated Nov 24 23:55:28.266 
 local adjacency to GigabitEthernet0/0/0/1

 Prefix Len 32, traffic index 0, precedence n/a, priority 4
  gateway array (0x89788098) reference count 1, flags 0x2010, source rib (7), 0 backups
                [1 type 3 flags 0x68501 (0x881ad880) ext 0x218 (0x8937a368)]
  LW-LDI[type=0, refc=0, ptr=0x0, sh-ldi=0x0]
  gateway array update type-time 1 Nov 24 23:55:28.266
 LDI Update time Nov 24 23:55:28.274
   via 2001:db8:10::3/128, 3 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0x8723e898 0x0]
    next hop VRF - 'default', table - 0xe0800000
    next hop 2001:db8:10::3/128 via 2001:db8:10::3/128

    Load distribution: 0 (refcount 1)

    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/1    fe80::a8c1:abff:fe93:753c
```
The IPv6 next-hop it recurses through does have the label:
```
RP/0/RP0/CPU0:pe1#show cef ipv6 2001:db8:10::3/128 detail
2001:db8:10::3/128, version 26, labeled SR, internal 0x1000001 0x8110 (ptr 0x8723e898) [3], 0x600 (0x884c8918), 0xa28 (0x8937a318)
 Updated Nov 24 23:52:18.643 
 local adjacency to GigabitEthernet0/0/0/1

 Prefix Len 128, traffic index 0, precedence n/a, priority 1
  gateway array (0x8835cd98) reference count 6, flags 0x68, source rib (7), 0 backups
                [3 type 5 flags 0x8401 (0x893b54b8) ext 0x0 (0x0)]
  LW-LDI[type=5, refc=3, ptr=0x884c8918, sh-ldi=0x893b54b8]
  gateway array update type-time 1 Nov 24 23:42:10.463
 LDI Update time Nov 24 23:42:10.463
 LW-LDI-TS Nov 24 23:42:10.463
 Accounting: Disabled
   via fe80::a8c1:abff:fe93:753c/128, GigabitEthernet0/0/0/1, 7 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x20002 [0x876f61e0 0x0]
    next hop fe80::a8c1:abff:fe93:753c/128
    local adjacency
     local label 16003      labels imposed {16003}

    Load distribution: 0 (refcount 1)

    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/1    fe80::a8c1:abff:fe93:753c
```
IPv6 works:
```
root@ce1:~# ping -c 3 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=1.80 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.81 ms
64 bytes from 2001:db8:187::3: icmp_seq=3 ttl=61 time=1.86 ms

--- 2001:db8:187::3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 1.804/1.824/1.859/0.024 ms
```

## Conclusion

The problem on XR is at the ingress PE, so enabling IPv4 on the egress side can't help.  XR doesn't inherit the SR label from the IPv6 next-hop into the IPv4 route's forwarding entry.  With IPv4 enabled on the outgoing interface it now picks the MPLS encapsulation from the adjacency but pushes zero labels, which looks like a bug rather than a design choice.  This is XRd control plane, whose software dataplane may differ from hardware platforms, so it may be worth checking on a real XR box or a later release.

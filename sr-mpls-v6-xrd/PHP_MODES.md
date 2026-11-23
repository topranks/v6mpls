# sr-mpls-v6-xrd: penultimate hop options

Results of testing the three label options for the last hop (P to PE2) in the XRd control-plane SR-MPLS lab.

| Mode | PE config (`router isis IGP / interface Loopback0 / address-family ipv6 unicast`) | Prefix-SID flags | P outgoing label for 16003 | IPv6 ping | IPv4 ping |
|---|---|---|---|---|---|
| UHP / Explicit Null | `prefix-sid index N explicit-null` | P:1 E:1 | Exp-Null-v6 (2) | OK | Fail (not sent by PE1) |
| UHP / non-null label | `prefix-sid index N php-disable` | P:1 E:0 | 16003 | OK | Fail (not sent by PE1) |
| PHP / Implicit Null | `prefix-sid index N` | P:0 E:0 | Pop | OK | Fail (not sent by PE1) |

N is 1 on PE1 and 3 on PE2.  The change was made on both PEs in each case.

PE1's forwarding for the IPv4 route from CE2 is the same in all three modes.  It is recursive via PE2's IPv6 loopback, with the "next hop VRF" in the default IPv6 table:
```
RP/0/RP0/CPU0:pe1#show cef ipv4 203.0.113.2/32 detail
203.0.113.2/32, version 7, internal 0x5000001 0x40 (ptr 0x8721bea0) [1], 0x0 (0x0), 0x0 (0x0)
 Updated Nov 23 22:39:14.337 
 local adjacency to GigabitEthernet0/0/0/1

 Prefix Len 32, traffic index 0, precedence n/a, priority 4
  gateway array (0x89788098) reference count 1, flags 0x2010, source rib (7), 0 backups
                [1 type 3 flags 0x68501 (0x881ad880) ext 0x218 (0x8937a368)]
  LW-LDI[type=0, refc=0, ptr=0x0, sh-ldi=0x0]
  gateway array update type-time 1 Nov 23 22:39:14.337
 LDI Update time Nov 23 22:39:14.345
   via 2001:db8:10::3/128, 3 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0x87256898 0x0]
    next hop VRF - 'default', table - 0xe0800000
    next hop 2001:db8:10::3/128 via 2001:db8:10::3/128

    Load distribution: 0 (refcount 1)

    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/1    fe80::a8c1:abff:feb8:a951
```
PE1 imposes 16003 towards P in all three modes (PE1 is not the penultimate hop, so the PHP setting does not change its outgoing label):
```
RP/0/RP0/CPU0:pe1#show mpls forwarding labels 16003 detail
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  16003       SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:feb8:a951   \
                                                                   1890        
     Updated: Nov 23 22:37:12.327
     Version: 21, Priority: 1
     Label Stack (Top -> Bottom): { 16003 }
     NHID: 0x20002, Encap-ID: N/A, Path idx: 0, Backup path idx: 0, Weight: 0
     MAC/Encaps: 14/18, MTU: 1500
     Outgoing Interface: GigabitEthernet0/0/0/1 (ifhandle 0x00000020)
     Packets Switched: 20
```

Captures are in [pcaps](pcaps), taken in the P container's netns on Gi0-0-0-0 (facing PE1) or Gi0-0-0-1 (facing PE2).  The capture filter only matches ICMP / ICMPv6 echo, labelled or not.

## UHP / non-null label

Config on both PEs (PE2 shown):
```
router isis IGP
 interface Loopback0
  address-family ipv6 unicast
   prefix-sid index 3 php-disable
```
PE2 advertises P:1, E:0:
```
RP/0/RP0/CPU0:pe2#show isis database pe2.00-00 verbose

IS-IS IGP (Level-2) Link State Database
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime/Rcvd  ATT/P/OL
pe2.00-00           * 0x00000006   0x42f0        3993 /*            0/0/0
  Area Address:   49.0001.0000
  LSP MTU:        1492
  NLPID:          0x8e
  MT:             IPv6 Unicast                                 0/0/0
  IPv6 Address:   2001:db8:10::3
  Hostname:       pe2
  Router Cap:     0.0.0.0 D:0 S:0
    IPv6 Router ID: 2001:db8:10::3
    Segment Routing: I:0 V:1, SRGB Base: 16000 Range: 8000
    Node Maximum SID Depth: 
      Label Imposition: 10
    SR Algorithm: 
      Algorithm: 0
      Algorithm: 1
  Metric: 100        MT (IPv6 Unicast) IS-Extended p.00
    Local Interface ID: 4, Remote Interface ID: 5
    Interface IPv6 Address: 2001:db8:2::4
    Neighbor IPv6 Address: 2001:db8:2::5
    Physical BW: 1000000 kbits/sec
    ADJ-SID: F:1 B:0 V:1 L:1 S:0 P:0 weight:0 Adjacency-sid:24001
  Metric: 100        MT (IPv6 Unicast) IPv6 2001:db8:2::4/127
    Prefix Attribute Flags: X:0 R:0 N:0 E:0 A:0
  Metric: 1          MT (IPv6 Unicast) IPv6 2001:db8:10::3/128
    Prefix-SID Index: 3, Algorithm:0, R:0 N:1 P:1 E:0 V:0 L:0
    Prefix Attribute Flags: X:0 R:0 N:1 E:0 A:0
```
P swaps 16003 for 16003:
```
RP/0/RP0/CPU0:p#show mpls forwarding labels 16003 detail
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  16003       SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:fe57:7f56   \
                                                                   0           
     Updated: Nov 23 22:38:22.963
     Version: 20, Priority: 1
     Label Stack (Top -> Bottom): { 16003 }
     NHID: 0x20002, Encap-ID: N/A, Path idx: 0, Backup path idx: 0, Weight: 0
     MAC/Encaps: 14/18, MTU: 1500
     Outgoing Interface: GigabitEthernet0/0/0/1 (ifhandle 0x00000018)
     Packets Switched: 0
```
On PE2, local label 16003 is "Aggregate", which pops it and looks up the packet in a table:
```
RP/0/RP0/CPU0:pe2#show mpls forwarding labels 16003 detail
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  Aggregate   SR Pfx (idx 3)     default                      72          
     Updated: Nov 23 22:37:06.595
     Label Stack (Top -> Bottom): { }
     MAC/Encaps: 0/0, MTU: 0
     Packets Switched: 1
```
```
RP/0/RP0/CPU0:pe2#show cef mpls local-label 16003 eos detail
Label/EOS 16003/1, Label-type Unknown, version 9, labeled SR, internal 0x1000081 0x10 (ptr 0x872a0848) [1], 0x600 (0x882746d8), 0xa20 (0x8937a278)
 Updated Nov 23 22:37:06.592
 Prefix Len 21, traffic index 0, precedence n/a, priority 1
  gateway array (0x88108598) reference count 2, flags 0x20048, source lsd (5), 0 backups
                [2 type 2 flags 0x8401 (0x893b5460) ext 0x0 (0x0)]
  LW-LDI[type=2, refc=1, ptr=0x882746d8, sh-ldi=0x893b5460]
  gateway array update type-time 1 Nov 23 22:37:06.592
 LDI Update time Nov 23 22:37:06.592
 LW-LDI-TS Nov 23 22:37:06.592
   via ::/128, 0 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0x872bf178 0x877d9108]
    recursion-via-/128
    next hop ::/128
     local label 16003      labels imposed {ImplNull}

    Load distribution: 0 (refcount 2)

    Hash  OK  Interface                 Address
    0     Y   recursive                 Lookup in table
```
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=1.77 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.84 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.767/1.805/1.844/0.038 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1020ms
```
On the PE1 to P link ([srxrd_uhp_nonnull_pe1_to_p.pcap](pcaps/srxrd_uhp_nonnull_pe1_to_p.pcap)) only the four IPv6 packets are seen.  None of the IPv4 echo requests appear:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:84:15:c4, Dst: aa:c1:ab:b8:a9:51
MultiProtocol Label Switching Header, Label: 16003, Exp: 0, S: 1, TTL: 63
    0000 0011 1110 1000 0011 .... .... .... = MPLS Label: 16003 (0x03e83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6

Frame 2: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:b8:a9:51, Dst: aa:c1:ab:84:15:c4
MultiProtocol Label Switching Header, Label: 16001, Exp: 0, S: 1, TTL: 62
    0000 0011 1110 1000 0001 .... .... .... = MPLS Label: 16001 (0x03e81)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::3, Dst: 2001:db8:187::2
Internet Control Message Protocol v6

Frame 3: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:84:15:c4, Dst: aa:c1:ab:b8:a9:51
MultiProtocol Label Switching Header, Label: 16003, Exp: 0, S: 1, TTL: 63
    0000 0011 1110 1000 0011 .... .... .... = MPLS Label: 16003 (0x03e83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6

Frame 4: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:b8:a9:51, Dst: aa:c1:ab:84:15:c4
MultiProtocol Label Switching Header, Label: 16001, Exp: 0, S: 1, TTL: 62
    0000 0011 1110 1000 0001 .... .... .... = MPLS Label: 16001 (0x03e81)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::3, Dst: 2001:db8:187::2
Internet Control Message Protocol v6
```
From P to PE2 ([srxrd_uhp_nonnull_p_to_pe2.pcap](pcaps/srxrd_uhp_nonnull_p_to_pe2.pcap)) the IPv6 packet keeps label 16003:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:e1:94:95, Dst: aa:c1:ab:57:7f:56
MultiProtocol Label Switching Header, Label: 16003, Exp: 0, S: 1, TTL: 62
    0000 0011 1110 1000 0011 .... .... .... = MPLS Label: 16003 (0x03e83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```

## PHP / Implicit Null

Config on both PEs (PE2 shown):
```
router isis IGP
 interface Loopback0
  address-family ipv6 unicast
   prefix-sid index 3
```
PE2 advertises P:0, E:0:
```
RP/0/RP0/CPU0:pe2#show isis database pe2.00-00 verbose | include Prefix-SID
    Prefix-SID Index: 3, Algorithm:0, R:0 N:1 P:0 E:0 V:0 L:0
```
P pops (Imp-Null):
```
RP/0/RP0/CPU0:p#show mpls forwarding labels 16003 detail
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  Pop         SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:fe57:7f56   \
                                                                   0           
     Updated: Nov 23 22:45:15.420
     Version: 24, Priority: 1
     Label Stack (Top -> Bottom): { Imp-Null }
     NHID: 0x20002, Encap-ID: N/A, Path idx: 0, Backup path idx: 0, Weight: 0
     MAC/Encaps: 14/14, MTU: 1500
     Outgoing Interface: GigabitEthernet0/0/0/1 (ifhandle 0x00000018)
     Packets Switched: 0
```
```
RP/0/RP0/CPU0:p#show cef mpls local-label 16003 eos detail
Label/EOS 16003/1, Label-type Unknown, version 24, labeled SR, internal 0x1000001 0x8110 (ptr 0x872a00c8) [1], 0x600 (0x88c84b18), 0xa20 (0x8930b368)
 Updated Nov 23 22:45:15.418 
 local adjacency to GigabitEthernet0/0/0/1

 Prefix Len 21, traffic index 0, precedence n/a, priority 1
  gateway array (0x88b19298) reference count 3, flags 0x68, source rib (7), 0 backups
                [3 type 4 flags 0x8401 (0x89346670) ext 0x0 (0x0)]
  LW-LDI[type=2, refc=1, ptr=0x88c84b18, sh-ldi=0x89346670]
  gateway array update type-time 1 Nov 23 22:45:15.418
 LDI Update time Nov 23 22:45:15.422
 LW-LDI-TS Nov 23 22:45:15.422
   via fe80::a8c1:abff:fe57:7f56/128, GigabitEthernet0/0/0/1, 7 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x20002 [0x87cad3c0 0x87cad1e0]
    next hop fe80::a8c1:abff:fe57:7f56/128
    local adjacency
     local label 16003      labels imposed {ImplNull}

    Load distribution: 0 (refcount 3)

    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/1    fe80::a8c1:abff:fe57:7f56
```
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=1.69 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.68 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.678/1.685/1.692/0.007 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1023ms
```
From P to PE2 ([srxrd_implicit_null_p_to_pe2.pcap](pcaps/srxrd_implicit_null_p_to_pe2.pcap)) the IPv6 packets are sent unlabelled with Ethertype IPv6:
```
    1   0.000000 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x000f, seq=1, hop limit=62
    2   0.000543 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x000f, seq=1, hop limit=63 (request in 1)
    3   1.001745 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x000f, seq=2, hop limit=62
    4   1.002280 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x000f, seq=2, hop limit=63 (request in 3)
```
```
Frame 1: 118 bytes on wire (944 bits), 118 bytes captured (944 bits)
Ethernet II, Src: aa:c1:ab:e1:94:95, Dst: aa:c1:ab:57:7f:56
    Destination: aa:c1:ab:57:7f:56
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: aa:c1:ab:e1:94:95
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv6 (0x86dd)
    [Stream index: 0]
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```
I also ran a 3 packet IPv4 ping while capturing on the PE1 to P link.  Nothing at all was captured ([srxrd_implicit_null_v4_pe1_to_p.pcap](pcaps/srxrd_implicit_null_v4_pe1_to_p.pcap) is empty):
```
tcpdump: listening on Gi0-0-0-0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
0 packets captured
0 packets received by filter
0 packets dropped by kernel
[stopped after 180s]
```

## UHP / Explicit Null

This is the lab's startup config:
```
router isis IGP
 interface Loopback0
  address-family ipv6 unicast
   prefix-sid index 3 explicit-null
```
PE2 advertises P:1, E:1:
```
RP/0/RP0/CPU0:pe2#show isis database pe2.00-00 verbose | include Prefix-SID
    Prefix-SID Index: 3, Algorithm:0, R:0 N:1 P:1 E:1 V:0 L:0
```
P swaps 16003 for 2:
```
RP/0/RP0/CPU0:p#show mpls forwarding labels 16003 detail
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  Exp-Null-v6 SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:fe57:7f56   \
                                                                   147         
     Updated: Nov 23 22:52:12.486
     Version: 28, Priority: 1
     Label Stack (Top -> Bottom): { 2 }
     NHID: 0x20002, Encap-ID: N/A, Path idx: 0, Backup path idx: 0, Weight: 0
     MAC/Encaps: 14/18, MTU: 1500
     Outgoing Interface: GigabitEthernet0/0/0/1 (ifhandle 0x00000018)
     Packets Switched: 2
```
PE2 looks up label 2 in the IPv6 table (0xe08fffff):
```
RP/0/RP0/CPU0:pe2#show cef mpls local-label 2 eos detail
Label/EOS ipv6-explicit-null/1, Label-type Unknown, version 4, internal 0x1000081 0x0 (ptr 0x872a15e8) [1], 0x600 (0x88274558), 0xa20 (0x8937a098)
 Updated Nov 23 22:37:03.979
 Prefix Len 21, traffic index 0, precedence n/a, priority 0
  gateway array (0x88108298) reference count 2, flags 0x20048, source lsd (5), 0 backups
                [2 type 2 flags 0x8401 (0x893b50f0) ext 0x0 (0x0)]
  LW-LDI[type=2, refc=1, ptr=0x88274558, sh-ldi=0x893b50f0]
  gateway array update type-time 1 Nov 23 22:37:03.979
 LDI Update time Nov 23 22:37:03.991
 LW-LDI-TS Nov 23 22:37:03.991
   via ::/128, 0 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0x872bf108 0x877d9098]
    recursion-via-/128
    next hop VRF - '', table - 0xe08fffff
    next hop ::/128
     local label 2      labels imposed {ImplNull}

    Load distribution: 0 (refcount 2)

    Hash  OK  Interface                 Address
    0     Y   recursive                 Lookup in table
```
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=1.82 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.49 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.491/1.653/1.816/0.162 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1014ms
```
From P to PE2 ([srxrd_explicit_null_p_to_pe2.pcap](pcaps/srxrd_explicit_null_p_to_pe2.pcap)) the IPv6 packets carry label 2, and there are no IPv4 packets:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:e1:94:95, Dst: aa:c1:ab:57:7f:56
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```

## Summary

The PHP setting changes P's behaviour as expected (Swap 2 / Swap 16003 / Pop), and IPv6 works in all three modes.  But IPv4 fails the same way in every mode.  PE1 installs the IPv4 route recursively via the IPv6 loopback with label 16003, yet the IPv4 packets never leave PE1.  Because the failure happens at the ingress PE, the last-hop label option makes no difference on XRd.

We can't tell from this lab how PE2 would treat an IPv4 packet arriving with label 2 or 16003, since none ever reach it.  The control plane does show PE2 maps label 2 to the IPv6 table (0xe08fffff).  Label 16003 (non-null) is a pop-and-lookup with no table shown.

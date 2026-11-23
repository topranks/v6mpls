# sr-mpls-v6-arista: penultimate hop options

Results of testing the three label options for the last hop (P to PE2) in the Arista cEOS SR-MPLS lab (cEOSLab 4.36.3M).

| Mode | PE config (`interface Loopback0`) | Node SID flags | P label op for 900003 | IPv6 ping | IPv4 ping |
|---|---|---|---|---|---|
| UHP / Explicit Null | `node-segment ipv6 index N explicit-null` | P:1 E:1 | swap 2 | OK | OK |
| UHP / non-null label | `node-segment ipv6 index N no-php` | P:1 E:0 | swap 900003 | OK | OK |
| PHP / Implicit Null | `node-segment ipv6 index N` | P:0 E:0 | pop | OK | OK |

N is 1 on PE1 and 3 on PE2.  The change was made on both PEs in each case.  The SRGB on cEOS starts at 900000, so PE2's node SID label is 900003.

IPv4 works in all three modes.  The key detail is `payload autoDecide` on the LFIB entries, and `via I, auto` for PE2's own label in the non-null case.  Neither P nor PE2 assumes the payload type from the label or FEC; they look at the packet.

**NOTE:** cEOS-lab forwards in software, so this shows how EOS programs the forwarding entries, not necessarily what a given hardware platform does.

PE1 uses the same SR tunnel and label for the IPv4 and IPv6 routes from CE2 in all modes:
```
pe1#show ip route 203.0.113.2/32

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, O IA - OSPF inter area, O E1 - OSPF external type 1,
       O E2 - OSPF external type 2, O N1 - OSPF NSSA external type 1,
       O N2 - OSPF NSSA external type2, O3 - OSPFv3,
       O3 IA - OSPFv3 inter area, O3 E1 - OSPFv3 external type 1,
       O3 E2 - OSPFv3 external type 2,
       O3 N1 - OSPFv3 NSSA external type 1,
       O3 N2 - OSPFv3 NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

 B I      203.0.113.2/32 [200/0]
           via 2001:db8:10::3/128, IS-IS SR tunnel index 1
              via fe80::a8c1:abff:fe24:3d67, Ethernet2/1, label 900003
```
```
pe1#show ipv6 route 2001:db8:187::3/128

VRF: default
Routing entry for 2001:db8:187::3/128
Source Codes:
       C - connected, S - static, K - kernel, O3 - OSPFv3,
       O3 IA - OSPFv3 inter area, O3 E1 - OSPFv3 external type 1,
       O3 E2 - OSPFv3 external type 2,
       O3 N1 - OSPFv3 NSSA external type 1
       O3 N2 - OSPFv3 NSSA external type2, B - Other BGP Routes,
       A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

 B I      2001:db8:187::3/128 [200/0]
           via 2001:db8:10::3/128, IS-IS SR tunnel index 1
              via fe80::a8c1:abff:fe24:3d67, Ethernet2/1, label 900003
```

Captures are in [pcaps](pcaps), all taken in the P container's netns on eth2 (facing PE2).  The capture filter only matches ICMP / ICMPv6 echo, labelled or not.  The `show mpls lfib route` outputs below are trimmed to remove the long legend of via/source codes at the top.

## UHP / Explicit Null

This is the lab's startup config:
```
interface Loopback0
   node-segment ipv6 index 3 explicit-null
```
```
pe2#show isis segment-routing prefix-segments

System ID: pe2			Instance: 'IGP'
SR supported Data-plane: MPLS			SR Router ID: 203.0.113.103

Node: 3      Proxy-Node: 0      Prefix: 0       Total Segments: 3

Flag Descriptions: R: Re-advertised, N: Node Segment, P: no-PHP
                   E: Explicit-NULL, V: Value, L: Local, A: Proxy-Node attached
Segment status codes: * - Self originated Prefix, L1 - level 1, L2 - level 2, ! - SR-unreachable,
                      # - Some IS-IS next-hops are SR-unreachable
   Prefix                      SID   Label Type       Flags                        System ID       Level Protection           Algorithm   
   ------------------------- ----- ------- ---------- ---------------------------- --------------- ----- -------------------- -------------
   2001:db8:10::1/128            1  900001 Node       R:0 N:1 P:1 E:1 V:0 L:0      pe1             L2    unprotected          SPF         
   2001:db8:10::2/128            2  900002 Node       R:0 N:1 P:1 E:1 V:0 L:0      p               L2    unprotected          SPF         
*  2001:db8:10::3/128            3  900003 Node       R:0 N:1 P:1 E:1 V:0 L:0      pe2             L2    unprotected          SPF         
```
P swaps 900003 for 2:
```
p#show mpls lfib route 900003
MPLS forwarding table (Label [metric] Vias) - 1 routes 
MPLS next-hop resolution allow default route: False
...
 IP    900003   [1], 2001:db8:10::3/128
                via M, fe80::a8c1:abff:fec9:1473, swap 2
                 payload autoDecide, ttlMode uniform, apply egress-acl
                 interface Ethernet2/1
```
PE2 has no LFIB entry for its own 900003, just entries for the other nodes' SIDs.  Label 2 is handled as a reserved label:
```
pe2#show mpls lfib route
MPLS forwarding table (Label [metric] Vias) - 3 routes 
MPLS next-hop resolution allow default route: False
...
 IA    100000   [1]
                via M, fe80::a8c1:abff:fe72:b38d, pop
                 payload autoDecide, ttlMode uniform, apply egress-acl
                 interface Ethernet2/1
 IP    900001   [1], 2001:db8:10::1/128
                via M, fe80::a8c1:abff:fe72:b38d, swap 900001
                 payload autoDecide, ttlMode uniform, apply egress-acl
                 interface Ethernet2/1
 IP    900002   [1], 2001:db8:10::2/128
                via M, fe80::a8c1:abff:fe72:b38d, swap 2
                 payload autoDecide, ttlMode uniform, apply egress-acl
                 interface Ethernet2/1
```
Both pings work:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=4.24 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.81 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.806/3.024/4.242/1.218 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=4.64 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=2.28 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 2.280/3.461/4.642/1.181 ms
```
On the wire ([srarista_explicit_null_p_to_pe2.pcap](pcaps/srarista_explicit_null_p_to_pe2.pcap)) the IPv4 packet goes to PE2 with label 2, and PE2 accepts and routes it anyway:
```
    1   0.000000 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 122 Echo (ping) request id=0x0018, seq=1, hop limit=63
    2   0.000845 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x0018, seq=1, hop limit=63 (request in 1)
    3   0.999870 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 122 Echo (ping) request id=0x0018, seq=2, hop limit=63
    4   1.000403 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x0018, seq=2, hop limit=63 (request in 3)
    5   1.054732  203.0.113.1 → 203.0.113.2  ICMP 102 Echo (ping) request  id=0x0019, seq=1/256, ttl=63
    6   1.055804  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x0019, seq=1/256, ttl=63 (request in 5)
    7   2.054270  203.0.113.1 → 203.0.113.2  ICMP 102 Echo (ping) request  id=0x0019, seq=2/512, ttl=63
    8   2.055096  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x0019, seq=2/512, ttl=63 (request in 7)
```
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:72:b3:8d, Dst: aa:c1:ab:c9:14:73
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6

Frame 5: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:72:b3:8d, Dst: aa:c1:ab:c9:14:73
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol

Frame 6: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:c9:14:73, Dst: aa:c1:ab:72:b3:8d
MultiProtocol Label Switching Header, Label: 900001, Exp: 0, S: 1, TTL: 63
    1101 1011 1011 1010 0001 .... .... .... = MPLS Label: 900001 (0xdbba1)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 4, Src: 203.0.113.2, Dst: 203.0.113.1
Internet Control Message Protocol
```

## UHP / non-null label

Config on both PEs (PE2 shown):
```
interface Loopback0
   node-segment ipv6 index 3 no-php
```
PE2 (and PE1) now advertise P:1, E:0.  P still has explicit-null configured:
```
pe2#show isis segment-routing prefix-segments

System ID: pe2			Instance: 'IGP'
SR supported Data-plane: MPLS			SR Router ID: 203.0.113.103

Node: 3      Proxy-Node: 0      Prefix: 0       Total Segments: 3

Flag Descriptions: R: Re-advertised, N: Node Segment, P: no-PHP
                   E: Explicit-NULL, V: Value, L: Local, A: Proxy-Node attached
Segment status codes: * - Self originated Prefix, L1 - level 1, L2 - level 2, ! - SR-unreachable,
                      # - Some IS-IS next-hops are SR-unreachable
   Prefix                      SID   Label Type       Flags                        System ID       Level Protection           Algorithm   
   ------------------------- ----- ------- ---------- ---------------------------- --------------- ----- -------------------- -------------
   2001:db8:10::1/128            1  900001 Node       R:0 N:1 P:1 E:0 V:0 L:0      pe1             L2    unprotected          SPF         
   2001:db8:10::2/128            2  900002 Node       R:0 N:1 P:1 E:1 V:0 L:0      p               L2    unprotected          SPF         
*  2001:db8:10::3/128            3  900003 Node       R:0 N:1 P:1 E:0 V:0 L:0      pe2             L2    unprotected          SPF         
```
P swaps 900003 for 900003:
```
p#show mpls lfib route 900003
MPLS forwarding table (Label [metric] Vias) - 1 routes 
MPLS next-hop resolution allow default route: False
...
 IP    900003   [1], 2001:db8:10::3/128
                via M, fe80::a8c1:abff:fec9:1473, swap 900003
                 payload autoDecide, ttlMode uniform, apply egress-acl
                 interface Ethernet2/1
```
PE2 now installs its own label 900003 as an IP lookup in the default VRF, with the address family set to "auto":
```
pe2#show mpls lfib route 900003
MPLS forwarding table (Label [metric] Vias) - 1 routes 
MPLS next-hop resolution allow default route: False
...
 IP    900003   [0], 2001:db8:10::3/128
                via I, auto, vrf default
```
This is the behaviour described in section 8.4 of the 4PE draft: pop the label, then check the IP version to pick the table.  Compare Junos, which installs the same label as "to table inet6.0".

Both pings work:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=62 time=2.01 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=62 time=2.06 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.014/2.035/2.057/0.021 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=62 time=2.07 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=62 time=2.07 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.071/2.072/2.073/0.001 ms
```
Note the reply TTL is 62 here, not 61 as in the other two modes, so one hop fewer decrements the TTL on the round trip.  Return traffic towards PE1 also uses the non-null label, since PE1 was changed too.

On the wire ([srarista_uhp_nonnull_p_to_pe2.pcap](pcaps/srarista_uhp_nonnull_p_to_pe2.pcap)) both address families carry 900003 to PE2:
```
    1   0.000000 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 122 Echo (ping) request id=0x001a, seq=1, hop limit=63
    2   0.000661 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x001a, seq=1, hop limit=63 (request in 1)
    3   1.001201 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 122 Echo (ping) request id=0x001a, seq=2, hop limit=63
    4   1.001843 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x001a, seq=2, hop limit=63 (request in 3)
    5   1.046764  203.0.113.1 → 203.0.113.2  ICMP 102 Echo (ping) request  id=0x001b, seq=1/256, ttl=63
    6   1.047381  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x001b, seq=1/256, ttl=63 (request in 5)
    7   2.047863  203.0.113.1 → 203.0.113.2  ICMP 102 Echo (ping) request  id=0x001b, seq=2/512, ttl=63
    8   2.048537  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x001b, seq=2/512, ttl=63 (request in 7)
```
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:72:b3:8d, Dst: aa:c1:ab:c9:14:73
MultiProtocol Label Switching Header, Label: 900003, Exp: 0, S: 1, TTL: 62
    1101 1011 1011 1010 0011 .... .... .... = MPLS Label: 900003 (0xdbba3)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6

Frame 5: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:72:b3:8d, Dst: aa:c1:ab:c9:14:73
MultiProtocol Label Switching Header, Label: 900003, Exp: 0, S: 1, TTL: 62
    1101 1011 1011 1010 0011 .... .... .... = MPLS Label: 900003 (0xdbba3)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```

## PHP / Implicit Null

Config on both PEs (PE2 shown):
```
interface Loopback0
   node-segment ipv6 index 3
```
```
pe2#show isis segment-routing prefix-segments

System ID: pe2			Instance: 'IGP'
SR supported Data-plane: MPLS			SR Router ID: 203.0.113.103

Node: 3      Proxy-Node: 0      Prefix: 0       Total Segments: 3

Flag Descriptions: R: Re-advertised, N: Node Segment, P: no-PHP
                   E: Explicit-NULL, V: Value, L: Local, A: Proxy-Node attached
Segment status codes: * - Self originated Prefix, L1 - level 1, L2 - level 2, ! - SR-unreachable,
                      # - Some IS-IS next-hops are SR-unreachable
   Prefix                      SID   Label Type       Flags                        System ID       Level Protection           Algorithm   
   ------------------------- ----- ------- ---------- ---------------------------- --------------- ----- -------------------- -------------
   2001:db8:10::1/128            1  900001 Node       R:0 N:1 P:0 E:0 V:0 L:0      pe1             L2    unprotected          SPF         
   2001:db8:10::2/128            2  900002 Node       R:0 N:1 P:1 E:1 V:0 L:0      p               L2    unprotected          SPF         
*  2001:db8:10::3/128            3  900003 Node       R:0 N:1 P:0 E:0 V:0 L:0      pe2             L2    unprotected          SPF         
```
P pops, still with `payload autoDecide`:
```
p#show mpls lfib route 900003
MPLS forwarding table (Label [metric] Vias) - 1 routes 
MPLS next-hop resolution allow default route: False
...
 IP    900003   [1], 2001:db8:10::3/128
                via M, fe80::a8c1:abff:fec9:1473, pop
                 payload autoDecide, ttlMode uniform, apply egress-acl
                 interface Ethernet2/1
```
Both pings work:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=2.20 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.94 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.944/2.072/2.200/0.128 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=2.14 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=1.90 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.896/2.016/2.137/0.120 ms
```
On the wire ([srarista_implicit_null_p_to_pe2.pcap](pcaps/srarista_implicit_null_p_to_pe2.pcap)) the requests from P to PE2 are unlabelled: 118 bytes for IPv6 and 98 bytes for IPv4.  P sets the Ethertype from the payload, so the IPv4 packet goes out as 0x0800.  The replies from PE2 carry PE1's label:
```
    1   0.000000 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x001c, seq=1, hop limit=62
    2   0.000715 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x001c, seq=1, hop limit=63 (request in 1)
    3   1.001043 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x001c, seq=2, hop limit=62
    4   1.001789 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x001c, seq=2, hop limit=63 (request in 3)
    5   1.050121  203.0.113.1 → 203.0.113.2  ICMP 98 Echo (ping) request  id=0x001d, seq=1/256, ttl=62
    6   1.050735  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x001d, seq=1/256, ttl=63 (request in 5)
    7   2.051198  203.0.113.1 → 203.0.113.2  ICMP 98 Echo (ping) request  id=0x001d, seq=2/512, ttl=62
    8   2.051739  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x001d, seq=2/512, ttl=63 (request in 7)
```
```
Frame 5: 98 bytes on wire (784 bits), 98 bytes captured (784 bits)
Ethernet II, Src: aa:c1:ab:72:b3:8d, Dst: aa:c1:ab:c9:14:73
    Destination: aa:c1:ab:c9:14:73
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: aa:c1:ab:72:b3:8d
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
    [Stream index: 0]
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
This matches the "implicit null with P inspecting IP version" option described in the main README.  P uses the label to choose the egress interface and next-hop MAC, then checks the payload to set the Ethertype.  PE2 accepts an IPv4 frame on an interface that has no IPv4 address.

## Summary

Arista forwards IPv4 over the IPv6 SR LSP in every mode, because no node ties the payload type to the label:

* **Explicit null**: PE2 accepts IPv4 under label 2, which strictly should only carry IPv6 (RFC 3032), and routes it in the IPv4 table.
* **Non-null label**: PE2's own label is programmed as `via I, auto, vrf default`, i.e. pop then look up by IP version.  This is 4PE draft section 8.4.
* **Implicit null**: P's pop is `payload autoDecide`, so it sets the Ethertype from the packet's IP version.

# sr-mpls-v6-junos: penultimate hop options

Results of testing the three label options for the last hop (P to PE2) in the vjunos-router SR-MPLS lab.  Image is vJunos-router 25.4R1-S2.3.

| Mode | PE config (`protocols isis source-packet-routing`) | Node SID flags | P label op for 400003 | IPv6 ping | IPv4 ping |
|---|---|---|---|---|---|
| UHP / Explicit Null | `explicit-null` | P:1, E:1 | Swap 2 | OK | Fail (dropped at PE2) |
| UHP / non-null label | `ultimate-hop-popping` | P:1, E:0 | Swap 400003 | OK | Fail (dropped at PE2) |
| PHP / Implicit Null | neither | P:0, E:0 | Pop | OK | Fail (dropped at P) |

The config change was made on both PEs in each case.  PE1 is the same in all three modes, it pushes label 400003 (PE2's node SID) for both the IPv4 and IPv6 routes from CE2:
```
admin@pe1> show route 2001:db8:10::3 table inet6.3 

inet6.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:10::3/128 *[L-ISIS/14] 00:03:00, metric 201
                    >  to fe80::aac1:abff:fe96:471c via ge-0/0/1.0, Push 400003
```
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:02:35, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe96:471c via ge-0/0/1.0, Push 400003
```
```
admin@pe1> show route 2001:db8:187::3 table inet6.0 

inet6.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:187::3/128*[BGP/170] 00:02:35, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe96:471c via ge-0/0/1.0, Push 400003
```

Captures are in [pcaps](pcaps), all taken on the P router's tap interface facing PE2 (tap2) unless stated.

## UHP / Explicit Null

This is the lab's startup config:
```
set protocols isis source-packet-routing explicit-null
```
PE2 advertises its node SID with both P (no-PHP) and E (explicit-null) set:
```
admin@pe2> show isis database pe2.00-00 extensive | match "prefix|SID|flags" 
     P2P IPv6 Adj-SID:      16, Weight:   0, Flags: F-VL--
   V6 prefix: 2001:db8:2::4/127               Metric:      100 Internal Up
   V6 prefix: 2001:db8:10::3/128              Metric:        1 Internal Up
    NLPID: 0x83, Fixed length: 27 bytes, Version: 1, Sysid length: 0 bytes
      P2P IPV6 Adj-SID - Flags:0xb0(F:1,B:0,V:1,L:1,S:0,P:0), Weight:0, Label: 16
      P2P IPv6 Adj-SID:      16, Weight:   0, Flags: F-VL--
    IPv6 prefix: 2001:db8:10::3/128 Metric 1 Up
      Node SID, Flags: 0x70(R:0,N:1,P:1,E:1,V:0,L:0), Algo: SPF(0), Value: 3
    IPv6 prefix: 2001:db8:2::4/127 Metric 100 Up
    Router Capability:  Router ID 203.0.113.103, Flags: 0x00
      SPRING Capability - Flags: 0xc0(I:1,V:1), Range: 8000, SID-Label: 400000
```
P swaps 400003 for 2:
```
admin@p> show route table mpls.0 label 400003 exact 

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:03:00, metric 101
                    >  to fe80::aac1:abff:fef0:8e85 via ge-0/0/1.0, Swap 2
```
On PE2 there is no entry for its own label 400003.  Label 2 is looked up in inet6.0 (and label 0 in inet.0):
```
admin@pe2> show route table mpls.0 

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 00:04:48, metric 1
                       to table inet.0
0(S=0)             *[MPLS/0] 00:04:48, metric 1
                       to table mpls.0
1                  *[MPLS/0] 00:04:48, metric 1
                       Receive
2                  *[MPLS/0] 00:04:48, metric 1
                       to table inet6.0
2(S=0)             *[MPLS/0] 00:04:48, metric 1
                       to table mpls.0
13                 *[MPLS/0] 00:04:48, metric 1
                       Receive
16                 *[L-ISIS/14] 00:03:01, metric 0
                    >  to fe80::aac1:abff:fed7:6ae1 via ge-0/0/1.0, Pop      
16(S=0)            *[L-ISIS/14] 00:03:01, metric 0
                    >  to fe80::aac1:abff:fed7:6ae1 via ge-0/0/1.0, Pop      
400001             *[L-ISIS/14] 00:02:54, metric 201
                    >  to fe80::aac1:abff:fed7:6ae1 via ge-0/0/1.0, Swap 400001
400002             *[L-ISIS/14] 00:03:01, metric 101
                    >  to fe80::aac1:abff:fed7:6ae1 via ge-0/0/1.0, Swap 2
400002(S=0)        *[L-ISIS/14] 00:03:01, metric 101
                    >  to fe80::aac1:abff:fed7:6ae1 via ge-0/0/1.0, Pop
```
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=5.02 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=2.40 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.403/3.710/5.018/1.307 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1018ms
```
On the wire from P to PE2 ([srjunos_explicit_null_p_to_pe2.pcap](pcaps/srjunos_explicit_null_p_to_pe2.pcap)) the IPv6 echo request carries label 2:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:d7:6a:e1, Dst: aa:c1:ab:f0:8e:85
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```
The IPv4 echo request also goes out with label 2.  PE2 looks it up in inet6.0 and drops it:
```
Frame 5: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:d7:6a:e1, Dst: aa:c1:ab:f0:8e:85
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```

## UHP / non-null label

Config change on both PEs:
```
delete protocols isis source-packet-routing explicit-null
set protocols isis source-packet-routing ultimate-hop-popping
```
In the CLI this option is described as _"Set P bit for no-php in all Prefix SID advertisements"_.  PE2 now advertises P:1 and E:0:
```
admin@pe2> show isis database pe2.00-00 extensive | match "Node SID" 
      Node SID, Flags: 0x60(R:0,N:1,P:1,E:0,V:0,L:0), Algo: SPF(0), Value: 3
```
P now swaps 400003 for 400003 (same SRGB on all nodes):
```
admin@p> show route table mpls.0 label 400003 exact 

mpls.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:08, metric 101
                    >  to fe80::aac1:abff:fef0:8e85 via ge-0/0/1.0, Swap 400003
```
PE2 installs its own label 400003, which pops and looks up the packet in inet6.0.  So it does the same as label 2:
```
admin@pe2> show route table mpls.0 label 400003 

mpls.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:07, metric 0
                       to table inet6.0, Pop      
400003(S=0)        *[L-ISIS/14] 00:00:05, metric 0
                       to table mpls.0
```
In the forwarding table, 400003 and label 2 point at the same table-lookup next-hop, index 2:
```
admin@pe2> show route forwarding-table family mpls label 400003 
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
400003             user     0                    rtbl        2     4
400003(S=0)        user     0                    rtbl        3     5
```
```
admin@pe2> show route forwarding-table family mpls label 2 
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
2                  user     0                    rtbl        2     4
2(S=0)             user     0                    rtbl        3     5
```
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=2.79 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=2.48 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.477/2.632/2.788/0.155 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1017ms
```
On the wire ([srjunos_uhp_nonnull_p_to_pe2.pcap](pcaps/srjunos_uhp_nonnull_p_to_pe2.pcap)) both IPv6 and IPv4 go to PE2 with label 400003:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:d7:6a:e1, Dst: aa:c1:ab:f0:8e:85
MultiProtocol Label Switching Header, Label: 400003, Exp: 0, S: 1, TTL: 62
    0110 0001 1010 1000 0011 .... .... .... = MPLS Label: 400003 (0x61a83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```
```
Frame 5: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:d7:6a:e1, Dst: aa:c1:ab:f0:8e:85
MultiProtocol Label Switching Header, Label: 400003, Exp: 0, S: 1, TTL: 62
    0110 0001 1010 1000 0011 .... .... .... = MPLS Label: 400003 (0x61a83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
So the IPv4 packet reaches PE2 with a label that PE2 itself has tied to inet6.0, and it is dropped there, exactly like the explicit-null case.  The 4PE draft section 8.4 scenario (egress PE checks the IP version after popping) is not what Junos does.

## PHP / Implicit Null

Config change on both PEs:
```
delete protocols isis source-packet-routing ultimate-hop-popping
```
PE2 advertises P:0 and E:0:
```
admin@pe2> show isis database pe2.00-00 extensive | match "Node SID" 
      Node SID, Flags: 0x40(R:0,N:1,P:0,E:0,V:0,L:0), Algo: SPF(0), Value: 3
```
P now pops:
```
admin@p> show route table mpls.0 label 400003 exact 

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:07, metric 101
                    >  to fe80::aac1:abff:fef0:8e85 via ge-0/0/1.0, Pop
```
```
admin@p> show route table mpls.0 label 400003 exact extensive | match "Next hop|operation" 
                Next hop type: Router, Next hop index: 608
                Next hop: fe80::aac1:abff:fef0:8e85 via ge-0/0/1.0, selected
                Label operation: Pop
```
```
admin@p> show route forwarding-table family mpls label 400003 
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
400003             user     0 fe80::aac1:abff:fef0:8e85
                                                Pop        608     3 ge-0/0/1.0
400003(S=0)        user     0 fe80::aac1:abff:fef0:8e85
                                                Pop        609     3 ge-0/0/1.0
```
The PFE next-hop is `MPLS->IPv6`.  The egress L2 encap is fixed by the next-hop, not by the payload:
```
admin@p> request pfe execute target fpc0 command "show nhdb id 608 extensive" 
SENT: Ukern command: show nhdb id 608 extensive

   ID      Type      Interface    Next Hop Addr    Protocol       Encap     MTU               Flags  PFE internal Flags
-----  --------  -------------  ---------------  ----------  ------------  ----  ------------------  ------------------
  608   Unicast  ge-0/0/1.0     -                MPLS->IPv6      Ethernet  9178  0x0000000000200001  0x0000000000000002

BFD Session Id: 321

Flags:              0x0000000000200001
PFE internal flags: 0x0000000000000002
Interface: 335

Topo-link:
[pfe-0]: 0x11c0000000026000
    ModifyNH: Subcode=SetNH-Token(7),Desc=0x0,Data=0x26000,NextNH=0
 (pfeDest:0, TokenIdMode:0/  , VC memberId:0, isReass:0, token:0x260/608)


Dram Bytes     : 444
Flags          : 0x0000000000200001
Parent NHID    : 607
Route Table ID : 0
PreComputed MTU: 0
Tunnel Self ID     : 0x0
Tunnel Parent ID   : 0x0
Parent NHID    : 607

NH Egress/Fabric:
        Feature List: NH
           [pfe-0]: 0x0802146000020000;
          f_mask:0x4014000000000; c_mask:0xe000000000000000; f_num:29; c_num:3,  inst:0x0
        Idx#13    counter:
           [pfe-0]: 0x2bfffffd48012900

        Idx#23 l3-mpls-decaps:
           [pfe-0]: 0x23fffffc0000002e

        Idx#25      ucast:
           [pfe-0]: 0x129802f920009005

(encap-ptr chain output truncated)
```
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=3.42 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=2.30 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.304/2.864/3.424/0.560 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1024ms
```
From P to PE2 ([srjunos_implicit_null_p_to_pe2.pcap](pcaps/srjunos_implicit_null_p_to_pe2.pcap)) only the IPv6 packets are seen, sent as plain IPv6:
```
    1   0.000000 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x0008, seq=1, hop limit=62
    2   0.001116 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x0008, seq=1, hop limit=63 (request in 1)
    3   0.999774 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x0008, seq=2, hop limit=62
    4   1.000637 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x0008, seq=2, hop limit=63 (request in 3)
```
```
Frame 1: 118 bytes on wire (944 bits), 118 bytes captured (944 bits)
Ethernet II, Src: aa:c1:ab:d7:6a:e1, Dst: aa:c1:ab:f0:8e:85
    Destination: aa:c1:ab:f0:8e:85
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: aa:c1:ab:d7:6a:e1
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv6 (0x86dd)
    [Stream index: 0]
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```
The IPv4 packet reaches P from PE1 with label 400003 ([srjunos_implicit_null_v4_pe1_to_p.pcap](pcaps/srjunos_implicit_null_v4_pe1_to_p.pcap), captured on P's tap1):
```
Frame 1: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:2f:7e:fb, Dst: aa:c1:ab:96:47:1c
MultiProtocol Label Switching Header, Label: 400003, Exp: 0, S: 1, TTL: 63
    0110 0001 1010 1000 0011 .... .... .... = MPLS Label: 400003 (0x61a83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
But it is never sent on to PE2.  The PFE "Normal discard" counter on P goes up by one for each IPv4 ping sent (27 before, 32 after a 5 packet ping):
```
admin@p> show pfe statistics traffic | match "discard|drop|error" 
    Software input control plane drops  :                    0
    Software input high drops           :                    0
    Software input medium drops         :                    0
    Software input low drops            :                    0
    Software output drops               :                    0
    Hardware input drops                :                    0
Packet Forwarding Engine hardware discard statistics:
    Data error                 :                    0
    TCP header length error    :                    0
    Normal discard             :                   27
    Extended discard           :                    0
    Info cell drops            :                    0
    Fabric drops               :                    0
Packet Forwarding Engine Input IPv4 Header Checksum Error and Output MTU Error statistics:
```
```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4092ms
```
```
admin@p> show pfe statistics traffic | match "Normal discard" 
    Normal discard             :                   32
```

## Summary

IPv4 fails in all three modes on Junos.  The LSP is built for an IPv6 FEC (PE2's loopback), and every node treats traffic on it as IPv6:

* **Explicit null**: P swaps to 2.  PE2 maps label 2 to inet6.0 and the IPv4 lookup fails.
* **Non-null label**: P swaps to PE2's own SID label.  PE2 programs that label as pop + lookup in inet6.0, using the same next-hop as label 2, so the result is the same as explicit null.
* **Implicit null**: P pops, but the pop next-hop is `MPLS->IPv6`.  The IPv4 payload is discarded in P's PFE and never reaches PE2.

In none of these cases does a Junos box look at the IP version nibble of the payload after removing the last label.

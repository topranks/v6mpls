# ldp-mpls-v6-junos: BGP labelled-unicast for IPv4

Testing whether BGP labelled-unicast (RFC 8277, SAFI 4) between the PEs can carry IPv4 over the IPv6-only LDP core.  This is the approach in sections 8.1-8.3 of the [4PE draft](https://www.ietf.org/archive/id/draft-mishra-idr-v4-islands-v6-core-4pe-10.html).  The IPv4 routes stay in the global inet.0 table; no VPN address families or routing instances are used.  Image is vJunos-router 25.4R1-S2.3.

The BGP side behaves exactly as in the SR lab, so see [../sr-mpls-v6-junos/BGP_LU_NOTES.md](../sr-mpls-v6-junos/BGP_LU_NOTES.md) for the detail on configuration options, label allocation (per prefix) and label 0 (connected routes only).  This file covers what differs with LDP as the transport.  It also covers the `family inet` test.

## Summary

* **BGP-LU works over LDP** with the same workaround as the SR lab: `local-ipv4-address` plus an import policy rewriting the next-hop to the sending PE's IPv6 loopback.
* With LDP `explicit-null` configured, P **swaps the LDP label for 2 even when the BGP label is below it**.  The packet arrives at PE2 with label stack `[2, S=0][BGP label, S=1]`.  That's valid: PE2 pops label 2 (`2(S=0)` = look up the next label in mpls.0) and forwards on the BGP label.  This matches the label stack in 4PE draft section 8.2.  With SR, Junos instead installs a separate `(S=0)` pop entry on P, so only the BGP label reaches PE2.
* **Without BGP-LU**, adding `family inet` (no address) to the core interfaces makes plain IPv4 work with LDP implicit null, the same as the SR lab.

## BGP-LU config

PE1 (PE2 is the mirror image with 198.51.100.5 / 2001:db8:10::1):
```
delete protocols bgp group IBGP family inet unicast
set protocols bgp group IBGP family inet labeled-unicast local-ipv4-address 198.51.100.1
set policy-options policy-statement LU-NH-V6 term pe2 from neighbor 2001:db8:10::3
set policy-options policy-statement LU-NH-V6 term pe2 then next-hop 2001:db8:10::3
set protocols bgp group IBGP family inet labeled-unicast import LU-NH-V6
```
The route arrives with PE2's IPv4 address as next-hop and is rewritten to PE2's loopback by the import policy.  It resolves to the BGP label (18) under the LDP label for PE2's loopback (17):
```
admin@pe1> show route receive-protocol bgp 2001:db8:10::3 table inet.0 extensive 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
* 203.0.113.2/32 (1 entry, 1 announced)
     Accepted
     Route Label: 18
     Nexthop: 198.51.100.5
     MED: 0
     Localpref: 100
     AS path: 64498 ? 
     Entropy label capable, ELCv3, next hop field matches route next hop
```
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:00:04, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fec3:1aad via ge-0/0/1.0, Push 18, Push 17(top)
```
On PE2, label 18 pops and sends straight to CE2:
```
admin@pe2> show route table mpls.0 protocol vpn 

mpls.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

18                 *[VPN/170] 00:00:08
                    >  to 198.51.100.6 via ge-0/0/0.0, Pop
```
```
admin@pe2> show route advertising-protocol bgp 2001:db8:10::1 table inet.0 extensive | match "^\*|Label" 
* 203.0.113.2/32 (1 entry, 1 announced)
     Route Label: 18
     Entropy label capable, ELCv3
```
IPv4 works:
```
root@ce1:~# ping -c 3 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=6.52 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=3.10 ms
64 bytes from 203.0.113.2: icmp_seq=3 ttl=61 time=2.63 ms

--- 203.0.113.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 2.632/4.085/6.523/1.734 ms
```

## Label stack with LDP explicit-null

P has a single entry for label 17 (no separate `(S=0)` entry as with SR), which swaps to 2:
```
admin@p> show route table mpls.0 label 17 

mpls.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[LDP/9] 00:03:22, metric 1
                    >  to fe80::aac1:abff:fecf:4ef5 via ge-0/0/1.0, Swap 2
```
So from P to PE2 ([bgplu_ldpjunos_p_to_pe2_v4.pcap](pcaps/bgplu_ldpjunos_p_to_pe2_v4.pcap)) the IPv4 packet carries the IPv6 explicit null label (S=0) on top of PE2's BGP label:
```
1	aa:c1:ab:e9:bf:4e	2,18	0,1	203.0.113.1	203.0.113.2	Echo (ping) request  id=0x0034, seq=1/256, ttl=63
2	aa:c1:ab:cf:4e:f5	16,18	0,1	203.0.113.2	203.0.113.1	Echo (ping) reply    id=0x0034, seq=1/256, ttl=63 (request in 1)
3	aa:c1:ab:e9:bf:4e	2,18	0,1	203.0.113.1	203.0.113.2	Echo (ping) request  id=0x0034, seq=2/512, ttl=63
4	aa:c1:ab:cf:4e:f5	16,18	0,1	203.0.113.2	203.0.113.1	Echo (ping) reply    id=0x0034, seq=2/512, ttl=63 (request in 3)
```
```
Frame 1: 106 bytes on wire (848 bits), 106 bytes captured (848 bits)
Ethernet II, Src: aa:c1:ab:e9:bf:4e, Dst: aa:c1:ab:cf:4e:f5
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 0, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...0 .... .... = MPLS Bottom Of Label Stack: 0
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
MultiProtocol Label Switching Header, Label: 18, Exp: 0, S: 1, TTL: 63
    0000 0000 0000 0001 0010 .... .... .... = MPLS Label: 18 (0x00012)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
(Frames 2 and 4 are the replies from PE2 towards P: P's LDP label 16 for PE1's loopback, over PE1's BGP label 18.)

Here label 2 is not the bottom of the stack, so it is legitimately just "pop and continue".  PE2's `2(S=0)` entry does exactly that (`to table mpls.0`), then label 18 forwards the IPv4 packet to CE2.

## `family inet` on the core interfaces (no BGP-LU)

As in the SR lab, I reverted to plain `family inet unicast extended-nexthop` and added `family inet` (no address) to the core interfaces on PE1, P (both) and PE2.  I also removed LDP `explicit-null` from the PEs:
```
delete protocols bgp group IBGP family inet labeled-unicast
set protocols bgp group IBGP family inet unicast extended-nexthop
set interfaces ge-0/0/1 unit 0 family inet          (PE1, PE2)
set interfaces ge-0/0/0 unit 0 family inet          (P)
set interfaces ge-0/0/1 unit 0 family inet          (P)
delete protocols ldp explicit-null                  (PE1, PE2)
```
P pops label 17, and PE1 uses a single label:
```
admin@p> show route table mpls.0 label 17 

mpls.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[LDP/9] 00:00:10, metric 1
                    >  to fe80::aac1:abff:fecf:4ef5 via ge-0/0/1.0, Pop      
17(S=0)            *[LDP/9] 00:00:10, metric 1
                    >  to fe80::aac1:abff:fecf:4ef5 via ge-0/0/1.0, Pop
```
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:00:09, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fec3:1aad via ge-0/0/1.0, Push 17
```
IPv4 works, where it failed in [PHP_MODES.md](PHP_MODES.md) without `family inet`:
```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=4.57 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=3.14 ms
64 bytes from 203.0.113.2: icmp_seq=3 ttl=61 time=2.93 ms
64 bytes from 203.0.113.2: icmp_seq=4 ttl=61 time=3.20 ms
64 bytes from 203.0.113.2: icmp_seq=5 ttl=61 time=2.88 ms

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 2.875/3.341/4.569/0.625 ms
```
The SR lab notes show which interfaces need it (the penultimate hop's egress and the egress PE's ingress) and that explicit null (label 2 at the bottom of the stack) still fails.

# sr-mpls-v6-arista

IPv6 only MPLS core lab based on Arista cEOS (cEOSLab 4.36.3M), with two CE devices based on vanilla debian container image running FRR.

## TL;DR

| Mode | PE config (`interface Loopback0`) | P label op for 900003 | IPv6 | IPv4 |
|---|---|---|---|---|
| PHP / Implicit Null (lab default) | `node-segment ipv6 index N` | pop | OK | OK |
| UHP / non-null label | `node-segment ipv6 index N no-php` | swap 900003 | OK | OK |
| UHP / Explicit Null | `node-segment ipv6 index N explicit-null` | swap 2 | OK | OK |

N is 1 on PE1 and 3 on PE2.  The SRGB on cEOS starts at 900000, so PE2's node SID label is 900003.

It just works.  No node ties the payload type to the label, they all look at the packet.  Note cEOS-lab forwards in software, so this shows how EOS programs things, not necessarily what a given hardware platform does.

Captures referenced below are in [pcaps](pcaps), taken in P's netns on eth2 (facing PE2).  Output is summarised with:
```
tshark -r <file> -T fields -e frame.number -e eth.type -e mpls.label -e _ws.col.Source -e _ws.col.Destination -e _ws.col.Protocol
```
The `show mpls lfib route` outputs are trimmed to remove the long legend at the top.

## Tests

PE1 uses the same SR tunnel and label for the IPv4 and IPv6 routes from CE2 in all modes (route legend trimmed):
```
pe1#show ip route 203.0.113.2/32
...
 B I      203.0.113.2/32 [200/0]
           via 2001:db8:10::3/128, IS-IS SR tunnel index 1
              via fe80::a8c1:abff:fe24:3d67, Ethernet2/1, label 900003
```
```
pe1#show ipv6 route 2001:db8:187::3/128
...
 B I      2001:db8:187::3/128 [200/0]
           via 2001:db8:10::3/128, IS-IS SR tunnel index 1
              via fe80::a8c1:abff:fe24:3d67, Ethernet2/1, label 900003
```

### PHP / Implicit Null

PE2 advertises P:0 E:0 (P still had explicit-null on its own SID when this was captured, ignore that; flag legend trimmed):
```
pe2#show isis segment-routing prefix-segments
...
   Prefix                      SID   Label Type       Flags                        System ID       Level Protection           Algorithm   
   ------------------------- ----- ------- ---------- ---------------------------- --------------- ----- -------------------- -------------
   2001:db8:10::1/128            1  900001 Node       R:0 N:1 P:0 E:0 V:0 L:0      pe1             L2    unprotected          SPF         
   2001:db8:10::2/128            2  900002 Node       R:0 N:1 P:1 E:1 V:0 L:0      p               L2    unprotected          SPF         
*  2001:db8:10::3/128            3  900003 Node       R:0 N:1 P:0 E:0 V:0 L:0      pe2             L2    unprotected          SPF         
```
P pops.  The key bit is `payload autoDecide`:
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
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=2.14 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=1.90 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.896/2.016/2.137/0.120 ms
```

#### On the wire

P sends the requests to PE2 unlabelled, setting the Ethertype from the payload, 0x86dd for IPv6 and 0x0800 for IPv4.  PE2 is fine taking an IPv4 frame on an interface with no IPv4 address.  Replies carry PE1's label ([srarista_implicit_null_p_to_pe2.pcap](pcaps/srarista_implicit_null_p_to_pe2.pcap)):
```
1	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	900001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	900001	2001:db8:187::3	2001:db8:187::2	ICMPv6
5	0x0800		203.0.113.1	203.0.113.2	ICMP
6	0x8847	900001	203.0.113.2	203.0.113.1	ICMP
7	0x0800		203.0.113.1	203.0.113.2	ICMP
8	0x8847	900001	203.0.113.2	203.0.113.1	ICMP
```
This is the "implicit null with P inspecting IP version" option from the main README.

### UHP / non-null label

```
interface Loopback0
   node-segment ipv6 index 3 no-php
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
PE2 installs its own label as an IP lookup with the address family "auto", i.e. pop and check the IP version.  This is section 8.4 of the 4PE draft.  Compare Junos which shows "to table inet6.0" (though it turns out to work there too, with `family inet`):
```
pe2#show mpls lfib route 900003
MPLS forwarding table (Label [metric] Vias) - 1 routes 
MPLS next-hop resolution allow default route: False
...
 IP    900003   [0], 2001:db8:10::3/128
                via I, auto, vrf default
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
(TTL 62 rather than 61, one less hop decrements TTL on the round trip in this mode.)

#### On the wire

Both address families go to PE2 with 900003 ([srarista_uhp_nonnull_p_to_pe2.pcap](pcaps/srarista_uhp_nonnull_p_to_pe2.pcap)):
```
1	0x8847	900003	2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	900001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x8847	900003	2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	900001	2001:db8:187::3	2001:db8:187::2	ICMPv6
5	0x8847	900003	203.0.113.1	203.0.113.2	ICMP
6	0x8847	900001	203.0.113.2	203.0.113.1	ICMP
7	0x8847	900003	203.0.113.1	203.0.113.2	ICMP
8	0x8847	900001	203.0.113.2	203.0.113.1	ICMP
```

### UHP / Explicit Null

```
interface Loopback0
   node-segment ipv6 index 3 explicit-null
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
PE2 has no LFIB entry for 900003 or 2, label 2 is handled as a reserved label.  IPv4 works anyway:
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=4.64 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=2.28 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 2.280/3.461/4.642/1.181 ms
```

#### On the wire

IPv4 under label 2 with S=1 (frames 5 and 7), which strictly speaking should only ever carry IPv6 (RFC 3032).  Arista doesn't care and routes it anyway ([srarista_explicit_null_p_to_pe2.pcap](pcaps/srarista_explicit_null_p_to_pe2.pcap)):
```
1	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	900001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	900001	2001:db8:187::3	2001:db8:187::2	ICMPv6
5	0x8847	2	203.0.113.1	203.0.113.2	ICMP
6	0x8847	900001	203.0.113.2	203.0.113.1	ICMP
7	0x8847	2	203.0.113.1	203.0.113.2	ICMP
8	0x8847	900001	203.0.113.2	203.0.113.1	ICMP
```

## BGP labelled-unicast

Not needed here, but I tried it to compare.  The session negotiates IPv4 LU fine, but it's a non-runner for two reasons.

The `next-hop address-family ipv6 originate` knob used for IPv4 unicast (RFC 8950) doesn't exist under `address-family ipv4 labeled-unicast`:
```
> neighbor IBGP next-hop address-family ipv6 originate

% Invalid input at line 4
```
So EOS has no next-hop it can send:
```
pe2#show bgp neighbors 2001:db8:10::1
...
Local next hop for next hop self:
  IPv4 with MPLS Labels: missing (cannot send updates with next hop self)
  IPv6 Unicast: 2001:db8:10::3
```
Giving it an IPv4 source interface (`next-hop-self source-interface Ethernet1/1`) gets the routes sent, but with an IPv4 next-hop that can't resolve in the core.  A route-map with `set ipv6 next-hop` didn't change that either:
```
pe1#show bgp ipv4 labeled-unicast detail
BGP routing table information for VRF default
Router identifier 203.0.113.101, local AS number 64496
BGP routing table entry for 198.51.100.4/30
 Paths: 1 available
  Local
    198.51.100.5 labels [ 3 ] from 2001:db8:10::3 (203.0.113.103)
      Origin IGP, metric 0, localpref 100, IGP metric -, weight 0, tag 0
      Received 00:00:21 ago, invalid, internal
      Rx SAFI: MplsLabel
      Tunnel RIB eligible
```
Secondly, `network 203.0.113.2/32` under the LU family didn't pick up CE2's loopback, as it's learned over eBGP IPv4 unicast.  Only the connected route above made it in.  More in [BGP_LU_NOTES.md](BGP_LU_NOTES.md).

# sr-mpls-v6-junos: BGP labelled-unicast for IPv4

Testing whether BGP labelled-unicast (RFC 8277, SAFI 4) between the PEs can carry IPv4 over the IPv6-only SR-MPLS core.  This is the approach in sections 8.1-8.3 of the [4PE draft](https://www.ietf.org/archive/id/draft-mishra-idr-v4-islands-v6-core-4pe-10.html).  The IPv4 routes stay in the global inet.0 table; no VPN address families or routing instances are used.  Image is vJunos-router 25.4R1-S2.3.

There is also an important unexpected finding at the end: [Addendum: `family inet` on core interfaces](#addendum-family-inet-on-core-interfaces).

## Summary

* **IPv4 over BGP-LU works**, but only with a workaround for the next-hop.  Junos won't send IPv4 labelled-unicast routes with an IPv6 next-hop.  There is no `extended-nexthop` option under `family inet labeled-unicast`.  It insists on a `local-ipv4-address`, which it uses as the BGP next-hop, and that can't be resolved in an IPv6-only core.  An import policy on the receiving PE that rewrites the next-hop to the sending PE's IPv6 loopback fixes it.  The route then resolves over the SR LSP in inet6.3, giving a two-label stack: SR transport label on top, BGP label at the bottom.
* **Label allocation is per prefix** for routes learned from the CE over eBGP, the "worst case".  `label-allocation per-nexthop` / `per-table` in the export policy and `aggregate-label` had no effect.
* **Label 0 (IPv4 explicit null)** can only be advertised for **connected** routes, with `explicit-null connected-only`.  There is no option to advertise label 0 for BGP-learned routes.
* **Label 0 only worked after adding `family inet`** (no address) on PE2's core interface.  Without it PE2 discarded the packet after popping label 0.  The per-prefix labels don't need this, because the egress PE pops the label and sends the packet straight to the CE without an IP lookup.

## RFC 8950 and RFC 8277

The standards themselves are compatible.  RFC 8950 (which obsoletes RFC 5549) defines IPv4 NLRI with an IPv6 next-hop for several AFI/SAFIs, including labelled-unicast (AFI 1 / SAFI 4).  RFC 8277 only defines the label encoding in the NLRI.  The limitation found here is in Junos: it supports `extended-nexthop` for `family inet unicast` but not for `family inet labeled-unicast`.

## Config: enabling labelled-unicast

Swapping `family inet unicast extended-nexthop` for labelled-unicast fails to commit without a local IPv4 address:
```
admin@pe1# set protocols bgp group IBGP family inet labeled-unicast 
COMMIT FAILED: Commit failed with the following errors:

[edit protocols]
  'bgp'
    Error in neighbor 2001:db8:10::3 of group IBGP:
Local IPv4 address must be configured under non-IPv6 NLRI with an IPv6 neighbor
error: configuration check-out failed
```
And there is no `extended-nexthop` option:
```
admin@pe1#set protocols bgp group IBGP family inet labeled-unicast extended-nexthop
                                                                    ^
syntax error.
```
The full list of options is:
```
admin@pe1#set protocols bgp group IBGP family inet labeled-unicast ?
Possible completions:
  <[Enter]>            Execute this command
  accept-srv6-service  Accept SRv6 service
> accepted-prefix-limit  Limit maximum number of prefixes accepted from a peer
> add-path             Advertise multiple paths to peer
  advertise-srv6-service  Advertise SRv6 service
> aggregate-label      Aggregate labels of incoming routes with the same FEC
> aigp                 Allow sending and receiving of AIGP attribute
+ apply-groups         Groups from which to inherit configuration data
+ apply-groups-except  Don't inherit configuration data from these groups
  damping              Enable route flap damping
> defer-initial-multipath-build  Defer initial multipath build until EOR is received
> delay-route-advertisements  Delay route updates for this family until FIB-sync
> egress-protection    Egress router protection
> entropy-label        Use entropy label for entropy label capable BGP LSPs
> explicit-null        Advertise explicit null
+ export               Export policy
> graceful-restart     BGP graceful restart options
+ import               Import policy
  local-ipv4-address   Local IPv4 address
> loops                Allow local AS in received AS paths, peer scoped loop check
> nexthop-resolution   Configure nexthop resolution properties
  no-install           Dont install received routes in forwarding
> output-queue-priority  Default output-queue to assign updates to
  per-group-label      Advertise prefixes with unique labels per group
  per-prefix-label     Allocate a unique label to each advertised prefix
> prefix-limit         Limit maximum number of prefixes from a peer
> protection           Compute backup path for active nexthop failure
  resolve-vpn          Install received NLRI in inet.3 also
> rib                  Select table used by labeled unicast routes
> rib-group            Routing table group
  route-age-bgp-view   Maintain BGP route's age based on Update messages only
> route-refresh-priority  Default output-queue to assign route refreshes to
> send-community-type  Configure community types to be sent
> topology             Multi topology routing tables
> traffic-statistics   Collect statistics for BGP label-switched paths
> withdraw-priority    Default output-queue to assign withdrawn routes to
  |                    Pipe through a command
```
As a first test I used each PE's CE-facing IPv4 address as `local-ipv4-address` (PE1 198.51.100.1, PE2 198.51.100.5):
```
delete protocols bgp group IBGP family inet unicast
set protocols bgp group IBGP family inet labeled-unicast local-ipv4-address 198.51.100.1
```
The session negotiates inet-labeled-unicast:
```
admin@pe1> show bgp neighbor 2001:db8:10::3 | match "NLRI|State|Error" 
  Type: Internal    State: Established    Flags: <Sync>
  Last State: OpenConfirm   Last Event: RecvKeepAlive
  Last Error: Cease
  NLRI inet6-unicast policies: Export: [ NEXT-HOP-SELF ] 
  NLRI inet-labeled-unicast policies: Export: [ NEXT-HOP-SELF ] 
  Error: 'Cease' Sent: 1 Recv: 1
  I/O Session Thread: bgpio-0 State: Enabled
  NLRI for restart configured on peer: inet6-unicast inet-labeled-unicast
  NLRI advertised by peer: inet6-unicast inet-labeled-unicast
  NLRI for this session: inet6-unicast inet-labeled-unicast
  NLRI(s) enabled for color nexthop resolution: inet-labeled-unicast
  Entropy label NLRI: inet-labeled-unicast
    RIB State: BGP restart is complete
    Send state: in sync
    RIB State: BGP restart is complete
    Send state: in sync
  NLRI that restart is negotiated for: inet6-unicast inet-labeled-unicast
  NLRI of received end-of-rib markers: inet6-unicast inet-labeled-unicast
  NLRI of all end-of-rib markers sent: inet6-unicast inet-labeled-unicast
```
PE1 advertises CE1's loopback with label 17:
```
admin@pe1> show route advertising-protocol bgp 2001:db8:10::3 table inet.0 extensive 

inet.0: 4 destinations, 4 routes (3 active, 0 holddown, 1 hidden)
* 203.0.113.1/32 (1 entry, 1 announced)
 BGP group IBGP type Internal
     Route Label: 17
     Nexthop: Self
     Flags: Nexthop Change
     MED: 0
     Localpref: 100
     AS path: [64496] 64497 ? 
     Entropy label capable, ELCv3
```
But the route received from PE2 is hidden.  The next-hop is PE2's `local-ipv4-address`, 198.51.100.5, which is unresolvable (`PnhUnresolv`):
```
admin@pe1> show route receive-protocol bgp 2001:db8:10::3 table inet.0 hidden extensive 

inet.0: 4 destinations, 4 routes (3 active, 0 holddown, 1 hidden)
  203.0.113.2/32 (1 entry, 0 announced)
     Accepted
     Route Label: 17
     Nexthop: 198.51.100.5
     MED: 0
     Localpref: 100
     AS path: 64498 ? 
     Entropy label capable, ELCv3, next hop field matches route next hop
```
```
admin@pe1> show route 203.0.113.2 table inet.0 hidden extensive 

inet.0: 4 destinations, 4 routes (3 active, 0 holddown, 1 hidden)
203.0.113.2/32 (1 entry, 0 announced)
         BGP    Preference: 170/-101
                Next hop type: Unusable, Next hop index: 0
                Address: 0x8a61594
                Next-hop reference count: 1
                Kernel Table Id: 0
                Source: 2001:db8:10::3
                State: <Hidden Int Ext Changed>
                Local AS: 64496 Peer AS: 64496
                Age: 7 	Metric: 0 
                Validation State: unverified 
                Task: BGP_64496.2001:db8:10::3
                AS path: 64498 ? 
                Accepted
                Route Label: 17
                Localpref: 100
                Router ID: 203.0.113.103
                Thread: junos-main 
                Indirect next hops: 1
                        Protocol next hop: 198.51.100.5 ResolvState: PnhUnresolv TSP index: 5
                        Label operation: Push 17
                        Label TTL action: prop-ttl
                        Load balance label: Label 17: None; 
                        Indirect next hop: 0x0 - INH Session ID: 0, INH non-key opaque: 0x0, INH key opaque: 0x0
```

## Workaround: rewrite the next-hop on import

The labs already have `routing-options resolution rib inet.0 inet6-resolution-ribs inet6.3`, which lets IPv4 routes resolve via IPv6 next-hops in inet6.3.  So an import policy that sets the next-hop to the advertising PE's IPv6 loopback makes the route usable.  On PE1:
```
set policy-options policy-statement LU-NH-V6 term pe2 from neighbor 2001:db8:10::3
set policy-options policy-statement LU-NH-V6 term pe2 then next-hop 2001:db8:10::3
set protocols bgp group IBGP family inet labeled-unicast import LU-NH-V6
```
And the equivalent on PE2, pointing at 2001:db8:10::1.  The route then resolves with two labels, the BGP label at the bottom and PE2's SR node SID on top:
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:00:04, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe41:89ad via ge-0/0/1.0, Push 17, Push 400003(top)
```
On PE2 the BGP label points directly at CE2, with no IP lookup:
```
admin@pe2> show route table mpls.0 label 17 extensive 

mpls.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
17  (1 entry, 1 announced)
TSI:
KRT in-kernel 17     /52 -> {Pop      }
        *VPN    Preference: 170
                Next hop type: Router, Next hop index: 613
                Address: 0x8a68d94
                Next-hop reference count: 2, Next-hop session id: 320
                Kernel Table Id: 0
                Source: 198.51.100.6
                Next hop: 198.51.100.6 via ge-0/0/0.0, selected
                Label operation: Pop      
                Load balance label: Stitching point; 
                Label element ptr: 0x115e74d8
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 320
                State: <Active Int Ext>
                Local AS: 64496 
                Age: 36 
                Validation State: unverified 
                Task: BGP_RT_Background
                Announcement bits (1): 1-KRT 
                AS path: 64498 ? 
		Ref Cnt: 1
                Thread: junos-main
```
IPv4 now works:
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=6.36 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=2.72 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.722/4.539/6.356/1.817 ms
```

## Label allocation

To see how labels are allocated, I added a second loopback on CE2 (203.0.113.12/32), which CE2 announces to PE2.  PE2 allocates a separate label for each prefix, even though both have the same next-hop (CE2), and both labels do the same thing:
```
admin@pe2> show route advertising-protocol bgp 2001:db8:10::1 table inet.0 extensive | match "^\*|Label" 
* 203.0.113.2/32 (1 entry, 1 announced)
     Route Label: 17
     Entropy label capable, ELCv3
* 203.0.113.12/32 (1 entry, 1 announced)
     Route Label: 18
     Entropy label capable, ELCv3
```
```
admin@pe2> show route table mpls.0 label 17 

mpls.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[VPN/170] 00:03:12
                    >  to 198.51.100.6 via ge-0/0/0.0, Pop
```
```
admin@pe2> show route table mpls.0 label 18 

mpls.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

18                 *[VPN/170] 00:00:12
                    >  to 198.51.100.6 via ge-0/0/0.0, Pop
```
Things I tried that did **not** change this:

* An export policy on the IBGP labelled-unicast family with `then label-allocation per-nexthop`, and then `per-table`.  I reset the session to force reallocation; the labels were still per prefix (19 and 20).  These policy actions appear to apply only to L3VPN.
* `aggregate-label community X`, with an import policy tagging all routes from CE2 with that community.  Still per prefix (21 and 22) after a session reset.  The CLI describes it as _"Aggregate labels of incoming routes with the same FEC"_, so it's for received labelled routes, not routes from an unlabelled eBGP peer.
```
admin@pe2> show route advertising-protocol bgp 2001:db8:10::1 table inet.0 extensive | match "^\*|Label|Communities" 
* 203.0.113.2/32 (1 entry, 1 announced)
     Route Label: 21
     Communities: 64496:2
     Entropy label capable, ELCv3
* 203.0.113.12/32 (1 entry, 1 announced)
     Route Label: 22
     Communities: 64496:2
     Entropy label capable, ELCv3
```

## Label 0 (IPv4 explicit null)

The only explicit-null option requires `connected-only`:
```
admin@pe1# set protocols bgp group IBGP family inet labeled-unicast explicit-null ?
Possible completions:
  <[Enter]>            Execute this command
  connected-only       Advertise explicit null only for connected routes
  |                    Pipe through a command
```
```
[edit protocols bgp group IBGP family inet labeled-unicast]
  'explicit-null'
    Missing mandatory statement: 'connected-only'
error: commit failed: (missing mandatory statements)
```
To test it I exported PE2's connected CE2 link subnet into IBGP:
```
set policy-options policy-statement LU-EXPORT term connected from protocol direct
set policy-options policy-statement LU-EXPORT term connected from route-filter 198.51.100.4/30 exact
set policy-options policy-statement LU-EXPORT term connected then accept
set protocols bgp group IBGP family inet labeled-unicast export [ NEXT-HOP-SELF LU-EXPORT ]
set protocols bgp group IBGP family inet labeled-unicast explicit-null connected-only
```
The connected route is advertised with label 0, while the BGP-learned routes keep per-prefix labels.  Without `explicit-null` the connected route got label 3 (implicit null).
```
admin@pe2> show route advertising-protocol bgp 2001:db8:10::1 table inet.0 extensive | match "^\*|Label" 
* 198.51.100.4/30 (1 entry, 1 announced)
     Route Label: 0
     Entropy label capable, ELCv3
* 203.0.113.2/32 (1 entry, 1 announced)
     Route Label: 21
     Entropy label capable, ELCv3
* 203.0.113.12/32 (1 entry, 1 announced)
     Route Label: 22
     Entropy label capable, ELCv3
```
```
admin@pe1> show route table inet.0 protocol bgp 

inet.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

198.51.100.4/30    *[BGP/170] 00:09:18, localpref 100, from 2001:db8:10::3
                      AS path: I, validation-state: unverified
                    >  to fe80::aac1:abff:fe41:89ad via ge-0/0/1.0, Push 0, Push 400003(top)
203.0.113.1/32     *[BGP/170] 00:20:04, MED 0, localpref 100
                      AS path: 64497 ?, validation-state: unverified
                    >  to 198.51.100.2 via ge-0/0/0.0
203.0.113.2/32     *[BGP/170] 00:09:18, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe41:89ad via ge-0/0/1.0, Push 21, Push 400003(top)
203.0.113.12/32    *[BGP/170] 00:09:18, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe41:89ad via ge-0/0/1.0, Push 22, Push 400003(top)
```
The CE2 loopbacks were reachable, but at first the ping to CE2's link address (label 0) failed:
```
root@ce1:~# ping -c 2 -I 203.0.113.1 198.51.100.6
PING 198.51.100.6 (198.51.100.6) from 203.0.113.1 : 56(84) bytes of data.

--- 198.51.100.6 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1014ms
```
The packets arrived at PE2 with label 0 ([bgplu_srjunos_p_to_pe2_v4.pcap](pcaps/bgplu_srjunos_p_to_pe2_v4.pcap)), but PE2 never sent them to CE2.  A capture on PE2's CE-facing tap was empty, and PE2's PFE "Normal discard" counter went up by one per packet (114 to 119 for a 5-packet ping):
```
1	aa:c1:ab:13:f1:55	0	1	203.0.113.1	198.51.100.6	Echo (ping) request  id=0x0020, seq=1/256, ttl=63
2	aa:c1:ab:13:f1:55	0	1	203.0.113.1	198.51.100.6	Echo (ping) request  id=0x0020, seq=2/512, ttl=63
3	aa:c1:ab:13:f1:55	22	1	203.0.113.1	203.0.113.12	Echo (ping) request  id=0x0021, seq=1/256, ttl=63
4	aa:c1:ab:f7:a6:6e	400001,19	0,1	203.0.113.12	203.0.113.1	Echo (ping) reply    id=0x0021, seq=1/256, ttl=63 (request in 3)
5	aa:c1:ab:13:f1:55	22	1	203.0.113.1	203.0.113.12	Echo (ping) request  id=0x0021, seq=2/512, ttl=63
6	aa:c1:ab:f7:a6:6e	400001,19	0,1	203.0.113.12	203.0.113.1	Echo (ping) reply    id=0x0021, seq=2/512, ttl=63 (request in 5)
7	aa:c1:ab:13:f1:55	21	1	203.0.113.1	203.0.113.2	Echo (ping) request  id=0x0022, seq=1/256, ttl=63
8	aa:c1:ab:f7:a6:6e	400001,19	0,1	203.0.113.2	203.0.113.1	Echo (ping) reply    id=0x0022, seq=1/256, ttl=63 (request in 7)
9	aa:c1:ab:13:f1:55	21	1	203.0.113.1	203.0.113.2	Echo (ping) request  id=0x0022, seq=2/512, ttl=63
10	aa:c1:ab:f7:a6:6e	400001,19	0,1	203.0.113.2	203.0.113.1	Echo (ping) reply    id=0x0022, seq=2/512, ttl=63 (request in 9)
```
(`tshark -T fields -e frame.number -e eth.src -e mpls.label -e mpls.bottom -e ip.src -e ip.dst -e _ws.col.Info`, captured on P's tap facing PE2.)

PE2's forwarding table was correct (label 0 is a table lookup, and 198.51.100.6 resolves to CE2):
```
admin@pe2> show route forwarding-table family mpls label 0 
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
0                  user     0                    rtbl        1     3
0(S=0)             user     0                    rtbl        3     4
```
```
admin@pe2> show route forwarding-table destination 198.51.100.6 table default 
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
198.51.100.6/32    dest     1 aa:c1:ab:3d:1a:57  ucst      605     6 ge-0/0/0.0
```
The difference is that after popping label 0 the packet needs an IPv4 lookup, and the interface it arrived on (ge-0/0/1, facing P) has no `family inet`:
```
admin@pe2> show configuration interfaces ge-0/0/1 
description P:ge-0/0/1;
mtu 9192;
unit 0 {
    family iso;
    family inet6 {
        address 2001:db8:2::4/127;
    }
    family mpls;
}
```
Adding `family inet` with **no address** fixed it:
```
set interfaces ge-0/0/1 unit 0 family inet
```
```
root@ce1:~# ping -c 5 -I 203.0.113.1 198.51.100.6
PING 198.51.100.6 (198.51.100.6) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 198.51.100.6: icmp_seq=1 ttl=61 time=29.6 ms
64 bytes from 198.51.100.6: icmp_seq=2 ttl=61 time=6.60 ms
64 bytes from 198.51.100.6: icmp_seq=3 ttl=61 time=2.91 ms
64 bytes from 198.51.100.6: icmp_seq=4 ttl=61 time=3.05 ms
64 bytes from 198.51.100.6: icmp_seq=5 ttl=61 time=3.09 ms

--- 198.51.100.6 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4004ms
rtt min/avg/max/mdev = 2.910/9.050/29.607/10.371 ms
```
The discard counter stayed at 119.

## On the wire

From PE1 to P ([bgplu_srjunos_pe1_to_p_v4.pcap](pcaps/bgplu_srjunos_pe1_to_p_v4.pcap)) PE1 pushes two labels: the SR label for PE2 (400003) and PE2's BGP label (21, 22, or 0).  The replies arrive from P with only PE1's BGP label (19), because P pops the transport label:
```
1	aa:c1:ab:84:ed:79	400003,21	0,1	203.0.113.1	203.0.113.2	Echo (ping) request  id=0x0026, seq=1/256, ttl=63
2	aa:c1:ab:41:89:ad	19	1	203.0.113.2	203.0.113.1	Echo (ping) reply    id=0x0026, seq=1/256, ttl=63 (request in 1)
3	aa:c1:ab:84:ed:79	400003,22	0,1	203.0.113.1	203.0.113.12	Echo (ping) request  id=0x0027, seq=1/256, ttl=63
4	aa:c1:ab:41:89:ad	19	1	203.0.113.12	203.0.113.1	Echo (ping) reply    id=0x0027, seq=1/256, ttl=63 (request in 3)
5	aa:c1:ab:84:ed:79	400003,0	0,1	203.0.113.1	198.51.100.6	Echo (ping) request  id=0x0028, seq=1/256, ttl=63
6	aa:c1:ab:41:89:ad	19	1	198.51.100.6	203.0.113.1	Echo (ping) reply    id=0x0028, seq=1/256, ttl=63 (request in 5)
```
```
Frame 1: 106 bytes on wire (848 bits), 106 bytes captured (848 bits)
Ethernet II, Src: aa:c1:ab:84:ed:79, Dst: aa:c1:ab:41:89:ad
MultiProtocol Label Switching Header, Label: 400003, Exp: 0, S: 0, TTL: 63
    0110 0001 1010 1000 0011 .... .... .... = MPLS Label: 400003 (0x61a83)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...0 .... .... = MPLS Bottom Of Label Stack: 0
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
MultiProtocol Label Switching Header, Label: 21, Exp: 0, S: 1, TTL: 63
    0000 0000 0000 0001 0101 .... .... .... = MPLS Label: 21 (0x00015)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
Even though explicit-null is configured for the SR node SIDs, P pops the transport label rather than swapping it for 2 when there is another label below it.  Junos installs a separate `(S=0)` entry for this case:
```
admin@p> show route table mpls.0 label 400003 

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:05:06, metric 101
                    >  to fe80::aac1:abff:fef7:a66e via ge-0/0/1.0, Swap 2
400003(S=0)        *[L-ISIS/14] 00:05:06, metric 101
                    >  to fe80::aac1:abff:fef7:a66e via ge-0/0/1.0, Pop
```
So between P and PE2 only the BGP label remains, and PE2 knows from that label how to forward the IPv4 packet.

## Final working config

PE2 (PE1 is the same with addresses swapped, and without the connected-route export):
```
admin@pe2> show configuration protocols bgp group IBGP 
type internal;
local-address 2001:db8:10::3;
family inet {
    labeled-unicast {
        local-ipv4-address 198.51.100.5;
        import LU-NH-V6;
        export [ NEXT-HOP-SELF LU-EXPORT ];
        explicit-null connected-only;
    }
}
family inet6 {
    unicast;
}
export NEXT-HOP-SELF;
neighbor 2001:db8:10::1 {
    description PE1;
}
```
```
admin@pe2> show configuration policy-options 
policy-statement LU-EXPORT {
    term connected {
        from {
            protocol direct;
            route-filter 198.51.100.4/30 exact;
        }
        then accept;
    }
}
policy-statement LU-NH-V6 {
    term pe1 {
        from neighbor 2001:db8:10::1;
        then {
            next-hop 2001:db8:10::1;
        }
    }
}
policy-statement NEXT-HOP-SELF {
    then {
        next-hop self;
    }
}
```
Plus `set interfaces ge-0/0/1 unit 0 family inet` if label 0 is used.

## Addendum: `family inet` on core interfaces

The label 0 result raised a question: were the original failures in [PHP_MODES.md](PHP_MODES.md) also caused by the core interfaces lacking `family inet`?  I put the PEs back to plain `family inet unicast extended-nexthop` (no labelled-unicast).  Then I added `family inet` with no address to the core interfaces on all three routers and re-tested each PHP mode:
```
set interfaces ge-0/0/1 unit 0 family inet     (PE1, PE2)
set interfaces ge-0/0/0 unit 0 family inet     (P)
set interfaces ge-0/0/1 unit 0 family inet     (P)
```

| Mode | Without `family inet` on core (PHP_MODES.md) | With `family inet` on core |
|---|---|---|
| UHP / Explicit Null | Fail | **Fail** (dropped at PE2, PFE discards 128 to 133) |
| UHP / non-null label | Fail | **Works** |
| PHP / Implicit Null | Fail | **Works** |

### Implicit null

```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=4.10 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=2.74 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 2.744/3.423/4.102/0.679 ms
```
P still pops using a PFE next-hop marked `MPLS->IPv6`:
```
admin@p> show route forwarding-table family mpls label 400003 
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
400003             user     0 fe80::aac1:abff:fef7:a66e
                                                Pop        607     3 ge-0/0/1.0
400003(S=0)        user     0 fe80::aac1:abff:fef7:a66e
                                                Pop        609     3 ge-0/0/1.0
```
```
admin@p> request pfe execute target fpc0 command "show nhdb id 607" 
SENT: Ukern command: show nhdb id 607

   ID      Type      Interface    Next Hop Addr    Protocol       Encap     MTU               Flags  PFE internal Flags
-----  --------  -------------  ---------------  ----------  ------------  ----  ------------------  ------------------
  607   Unicast  ge-0/0/1.0     -                MPLS->IPv6      Ethernet  9178  0x0000000000200001  0x0000000000000002
```
But the IPv4 packet goes out to PE2 with the IPv4 Ethertype ([famlyinet_srjunos_implicit_null_p_to_pe2.pcap](pcaps/famlyinet_srjunos_implicit_null_p_to_pe2.pcap)):
```
    1   0.000000  203.0.113.1 → 203.0.113.2  ICMP 98 Echo (ping) request  id=0x002b, seq=1/256, ttl=62
    2   0.001306  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x002b, seq=1/256, ttl=63 (request in 1)
    3   0.999577  203.0.113.1 → 203.0.113.2  ICMP 98 Echo (ping) request  id=0x002b, seq=2/512, ttl=62
    4   1.000372  203.0.113.2 → 203.0.113.1  ICMP 102 Echo (ping) reply    id=0x002b, seq=2/512, ttl=63 (request in 3)
```
```
Frame 1: 98 bytes on wire (784 bits), 98 bytes captured (784 bits)
Ethernet II, Src: aa:c1:ab:13:f1:55, Dst: aa:c1:ab:f7:a6:6e
    Destination: aa:c1:ab:f7:a6:6e
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: aa:c1:ab:13:f1:55
        .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
    [Stream index: 0]
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
So the `MPLS->IPv6` label on the next-hop doesn't fix the egress Ethertype as I concluded in PHP_MODES.md.  The PFE sets the Ethertype from the payload, but only sends the packet if the egress interface has the matching family configured.

I then removed `family inet` from one interface at a time.  Both of these are needed:

* P's interface towards PE2 (removing it: 3/3 lost)
* PE2's interface towards P (removing it: 3/3 lost)

### UHP / non-null label

With `ultimate-hop-popping` on the PEs, P swaps 400003 to 400003 and PE2 still shows the label as "to table inet6.0":
```
admin@p> show route table mpls.0 label 400003 

mpls.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:12, metric 101
                    >  to fe80::aac1:abff:fef7:a66e via ge-0/0/1.0, Swap 400003
```
```
admin@pe2> show route table mpls.0 label 400003 

mpls.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:05, metric 0
                       to table inet6.0, Pop      
400003(S=0)        *[L-ISIS/14] 00:00:02, metric 0
                       to table mpls.0
```
But IPv4 works:
```
root@ce1:~# ping -c 3 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=11.4 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=21.2 ms
64 bytes from 203.0.113.2: icmp_seq=3 ttl=61 time=9.82 ms

--- 203.0.113.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 9.823/14.146/21.168/5.009 ms
```
```
1	aa:c1:ab:13:f1:55	400003	1	203.0.113.1	203.0.113.2	Echo (ping) request  id=0x0030, seq=1/256, ttl=63
2	aa:c1:ab:f7:a6:6e	400001	1	203.0.113.2	203.0.113.1	Echo (ping) reply    id=0x0030, seq=1/256, ttl=63 (request in 1)
```
([famlyinet_srjunos_uhp_nonnull_p_to_pe2.pcap](pcaps/famlyinet_srjunos_uhp_nonnull_p_to_pe2.pcap))

So the egress PE does pop the label and check the IP version, as in 4PE draft section 8.4, despite what the `to table inet6.0` display suggests.

### Explicit null

Label 2 is still a hard IPv6-only lookup on PE2.  With P swapping to 2, IPv4 fails and PE2's PFE "Normal discard" counter rises by one per packet:
```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4090ms
```
```
admin@pe2> show pfe statistics traffic | match "Normal discard" 
    Normal discard             :                  133
```
(It was 128 before this ping.)

### Takeaway

On vJunos the IPv4-over-IPv6-LSP failures with implicit null and non-null UHP come from IPv4 not being enabled on the core interfaces, not from the LSP's FEC.  With a bare `family inet` on the penultimate hop's egress and the egress PE's ingress interface, Junos behaves like Arista for those two modes.  Explicit null still fails, as label 2 is strictly IPv6.

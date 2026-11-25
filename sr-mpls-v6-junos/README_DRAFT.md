# sr-mpls-v6-junos

IPv6 only MPLS core lab based on vjunos-router, with two CE devices based on vanilla debian container image.

## vjunos-router

The Juniper vjunos-router images are available free of charge at the below URL:

https://support.juniper.net/support/downloads/?p=vjunos-router

## TL;DR

| Mode | PE config (`protocols isis source-packet-routing`) | P label op | IPv6 | IPv4 |
|---|---|---|---|---|
| PHP / Implicit Null (lab default) | neither knob | Pop | OK | OK* |
| UHP / non-null label | `ultimate-hop-popping` | Swap 400003 | OK | OK* |
| UHP / Explicit Null | `explicit-null` | Swap 2 | OK | Fail |

\* Only with `family inet` (no address) on the core interfaces.  Without it IPv4 fails in every mode.

The startup configs have implicit null and `family inet` on the core interfaces, so IPv4 works out of the box:
```
set interfaces ge-0/0/1 unit 0 family inet     (PE1, PE2)
set interfaces ge-0/0/0 unit 0 family inet     (P)
set interfaces ge-0/0/1 unit 0 family inet     (P)
```
Needed on P's interface towards the egress PE, and on the egress PE's interface towards P.  I added it everywhere so it works both directions.

Captures referenced below are in [pcaps](pcaps), and were taken on P's tap interfaces.  Output is summarised with:
```
tshark -r <file> -T fields -e frame.number -e eth.type -e mpls.label -e _ws.col.Source -e _ws.col.Destination -e _ws.col.Protocol
```

## Tests

PE1 learns both CE2 routes in IBGP with PE2's IPv6 loopback as next-hop, and pushes PE2's node SID label 400003 for both of them:
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
That's the same in all three modes, what changes is what P does with 400003.

### PHP / Implicit Null

PE2 advertises its node SID with P:0 and E:0:
```
admin@pe2> show isis database pe2.00-00 extensive | match "Node SID" 
      Node SID, Flags: 0x40(R:0,N:1,P:0,E:0,V:0,L:0), Algo: SPF(0), Value: 3
```
So P pops it:
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
Interestingly the PFE next-hop for the pop says `MPLS->IPv6`:
```
admin@p> request pfe execute target fpc0 command "show nhdb id 607" 
SENT: Ukern command: show nhdb id 607

   ID      Type      Interface    Next Hop Addr    Protocol       Encap     MTU               Flags  PFE internal Flags
-----  --------  -------------  ---------------  ----------  ------------  ----  ------------------  ------------------
  607   Unicast  ge-0/0/1.0     -                MPLS->IPv6      Ethernet  9178  0x0000000000200001  0x0000000000000002
```
But IPv4 works anyway:
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.
64 bytes from 203.0.113.2: icmp_seq=1 ttl=61 time=4.10 ms
64 bytes from 203.0.113.2: icmp_seq=2 ttl=61 time=2.74 ms

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 2.744/3.423/4.102/0.679 ms
```

#### On the wire

P to PE2, the echo requests leave P unlabelled with Ethertype IPv4 (0x0800), so P is setting the Ethertype from the payload regardless of what the next-hop says.  Replies from PE2 carry PE1's label 400001 ([famlyinet_srjunos_implicit_null_p_to_pe2.pcap](pcaps/famlyinet_srjunos_implicit_null_p_to_pe2.pcap)):
```
1	0x0800		203.0.113.1	203.0.113.2	ICMP
2	0x8847	400001	203.0.113.2	203.0.113.1	ICMP
3	0x0800		203.0.113.1	203.0.113.2	ICMP
4	0x8847	400001	203.0.113.2	203.0.113.1	ICMP
```
IPv6 is the same, just with Ethertype 0x86dd ([srjunos_implicit_null_p_to_pe2.pcap](pcaps/srjunos_implicit_null_p_to_pe2.pcap)):
```
1	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	400001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	400001	2001:db8:187::3	2001:db8:187::2	ICMPv6
```

#### Without `family inet`

Remove `family inet` from P's ge-0/0/1 or PE2's ge-0/0/1 and IPv4 breaks.  The packet arrives at P with label 400003 but never leaves, and P's PFE "Normal discard" counter goes up one per ping (27 before, 32 after a 5 packet ping):
```
admin@p> show pfe statistics traffic | match "Normal discard" 
    Normal discard             :                   32
```
So the PFE is fine sending an IPv4 packet with the IPv4 Ethertype, it just won't do so out an interface with no IPv4 family.  Same story on PE2 receiving it.

### UHP / non-null label

```
set protocols isis source-packet-routing ultimate-hop-popping
```
In the CLI this is described as _"Set P bit for no-php in all Prefix SID advertisements"_.  PE2 now sets P:1:
```
admin@pe2> show isis database pe2.00-00 extensive | match "Node SID" 
      Node SID, Flags: 0x60(R:0,N:1,P:1,E:0,V:0,L:0), Algo: SPF(0), Value: 3
```
P swaps 400003 for 400003 (same SRGB everywhere):
```
admin@p> show route table mpls.0 label 400003 

mpls.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:12, metric 101
                    >  to fe80::aac1:abff:fef7:a66e via ge-0/0/1.0, Swap 400003
```
PE2 installs its own label with a lookup in inet6.0, which doesn't look promising for IPv4:
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
So despite the "to table inet6.0" PE2 pops the label and checks the IP version, as in section 8.4 of the 4PE draft.

#### On the wire

IPv4 goes to PE2 with label 400003 ([famlyinet_srjunos_uhp_nonnull_p_to_pe2.pcap](pcaps/famlyinet_srjunos_uhp_nonnull_p_to_pe2.pcap)):
```
1	0x8847	400003	203.0.113.1	203.0.113.2	ICMP
2	0x8847	400001	203.0.113.2	203.0.113.1	ICMP
3	0x8847	400003	203.0.113.1	203.0.113.2	ICMP
4	0x8847	400001	203.0.113.2	203.0.113.1	ICMP
```

### UHP / Explicit Null

```
set protocols isis source-packet-routing explicit-null
```
```
admin@pe2> show isis database pe2.00-00 extensive | match "Node SID" 
      Node SID, Flags: 0x70(R:0,N:1,P:1,E:1,V:0,L:0), Algo: SPF(0), Value: 3
```
P swaps 400003 for 2:
```
admin@p> show route table mpls.0 label 400003 exact 

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:03:00, metric 101
                    >  to fe80::aac1:abff:fef0:8e85 via ge-0/0/1.0, Swap 2
```
On PE2 label 2 is hard-coded to a lookup in inet6.0:
```
admin@pe2> show route table mpls.0 label 2 exact 

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2                  *[MPLS/0] 03:50:54, metric 1
                       to table inet6.0
```
IPv6 works, IPv4 doesn't, even with `family inet`:
```
root@ce1:~# ping -c 5 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
5 packets transmitted, 0 received, 100% packet loss, time 4090ms
```
PE2's PFE "Normal discard" counter goes from 128 to 133:
```
admin@pe2> show pfe statistics traffic | match "Normal discard" 
    Normal discard             :                  133
```

#### On the wire

Both IPv6 and IPv4 go to PE2 with label 2 and S=1 ([srjunos_explicit_null_p_to_pe2.pcap](pcaps/srjunos_explicit_null_p_to_pe2.pcap)).  Frames 5 and 6 are the IPv4 echo requests, which get no reply:
```
1	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	400001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	400001	2001:db8:187::3	2001:db8:187::2	ICMPv6
5	0x8847	2	203.0.113.1	203.0.113.2	ICMP
6	0x8847	2	203.0.113.1	203.0.113.2	ICMP
```
To be fair to Juniper an IPv4 packet under label 2 at the bottom of the stack isn't valid, so dropping it is arguably correct.

## BGP labelled-unicast

Can we sidestep all this with BGP-LU, like 4PE draft sections 8.1 - 8.3?  Kind of, but it's not pretty.

Junos has no `extended-nexthop` option under `family inet labeled-unicast`, so it won't send IPv4 LU routes with an IPv6 next-hop.  It insists on a `local-ipv4-address` and uses that as the next-hop, which is unresolvable in an IPv6-only core.  The workaround is an import policy on the receiving PE rewriting the next-hop to the sender's IPv6 loopback:
```
set protocols bgp group IBGP family inet labeled-unicast local-ipv4-address 198.51.100.1
set policy-options policy-statement LU-NH-V6 term pe2 from neighbor 2001:db8:10::3
set policy-options policy-statement LU-NH-V6 term pe2 then next-hop 2001:db8:10::3
set protocols bgp group IBGP family inet labeled-unicast import LU-NH-V6
```
Then we get a two label stack, SR on top, BGP label at the bottom.  Here PE2 is also exporting its connected CE2 link with `explicit-null connected-only`:
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
PE1 to P ([bgplu_srjunos_pe1_to_p_v4.pcap](pcaps/bgplu_srjunos_pe1_to_p_v4.pcap)):
```
1	0x8847	400003,21	203.0.113.1	203.0.113.2	ICMP
2	0x8847	19	203.0.113.2	203.0.113.1	ICMP
3	0x8847	400003,22	203.0.113.1	203.0.113.12	ICMP
4	0x8847	19	203.0.113.12	203.0.113.1	ICMP
5	0x8847	400003,0	203.0.113.1	198.51.100.6	ICMP
6	0x8847	19	198.51.100.6	203.0.113.1	ICMP
```
That works, but:

* One term per remote PE, on every PE.  Doesn't scale, and gets worse with route reflectors.
* Every PE still needs an IPv4 address for `local-ipv4-address`, even if it's just a placeholder.
* Labels are allocated per prefix (21, 22 above), the worst case.  `label-allocation per-nexthop` / `per-table` and `aggregate-label` made no difference.
* Label 0 (explicit null) is only allowed as `explicit-null connected-only`, so only for connected routes (frame 5 above).  Not for routes learned from the CE.

All the gory detail is in [BGP_LU_NOTES.md](BGP_LU_NOTES.md).

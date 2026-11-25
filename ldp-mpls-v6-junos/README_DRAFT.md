# ldp-mpls-v6-junos

IPv6 only MPLS core lab based on vjunos-router, the same as [sr-mpls-v6-junos](../sr-mpls-v6-junos) but using LDP instead of SR-MPLS.  LDP runs over IPv6 transport with `family inet6` only, so the only FECs are the IPv6 loopbacks.  CE devices are vanilla debian containers running FRR.

## vjunos-router

The Juniper vjunos-router images are available free of charge at the below URL:

https://support.juniper.net/support/downloads/?p=vjunos-router

## TL;DR

| Mode | PE config (`protocols ldp`) | Label PE2 advertises | P label op | IPv6 | IPv4 |
|---|---|---|---|---|---|
| PHP / Implicit Null (lab default) | (default) | 3 | Pop | OK | OK* |
| UHP / non-null label | not possible | - | - | - | - |
| UHP / Explicit Null | `explicit-null` | 2 | Swap 2 | OK | Fail |

\* Only with `family inet` (no address) on the core interfaces, which the startup configs now have:
```
set interfaces ge-0/0/1 unit 0 family inet     (PE1, PE2)
set interfaces ge-0/0/0 unit 0 family inet     (P)
set interfaces ge-0/0/1 unit 0 family inet     (P)
```

Results are the same as the SR lab, how the label gets signalled doesn't matter much.  See the [SR lab README](../sr-mpls-v6-junos/README_DRAFT.md) for more detail on the Junos internals.

Captures referenced below are in [pcaps](pcaps), and were taken on P's tap2 (facing PE2).  Output is summarised with:
```
tshark -r <file> -T fields -e frame.number -e eth.type -e mpls.label -e _ws.col.Source -e _ws.col.Destination -e _ws.col.Protocol
```

## Tests

### PHP / Implicit Null

PE2 advertises label 3 for its loopback:
```
admin@pe2> show ldp database 
Input label database, 203.0.113.103:0--203.0.113.102:0
Labels received: 3
  Label     Prefix
     16      2001:db8:10::1/128
      2      2001:db8:10::2/128
     17      2001:db8:10::3/128

Output label database, 203.0.113.103:0--203.0.113.102:0
Labels advertised: 3
  Label     Prefix
     19      2001:db8:10::1/128
     18      2001:db8:10::2/128
      3      2001:db8:10::3/128
```
PE1 pushes P's label 17 for the IPv4 route from CE2:
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:00:09, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fec3:1aad via ge-0/0/1.0, Push 17
```
P pops 17:
```
admin@p> show route table mpls.0 label 17 

mpls.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[LDP/9] 00:00:10, metric 1
                    >  to fe80::aac1:abff:fecf:4ef5 via ge-0/0/1.0, Pop      
17(S=0)            *[LDP/9] 00:00:10, metric 1
                    >  to fe80::aac1:abff:fecf:4ef5 via ge-0/0/1.0, Pop
```
IPv4 works:
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

#### On the wire

I didn't save an IPv4 capture with `family inet` in this lab, but the SR lab shows the IPv4 packet going from P to PE2 unlabelled with Ethertype 0x0800.  Without `family inet` the IPv4 packet never leaves P, only IPv6 shows up, unlabelled with 0x86dd ([ldpjunos_implicit_null_p_to_pe2.pcap](pcaps/ldpjunos_implicit_null_p_to_pe2.pcap)):
```
1	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	16	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	16	2001:db8:187::3	2001:db8:187::2	ICMPv6
```
P's PFE "Normal discard" went from 23 to 25 for a 2 packet IPv4 ping in that case.

### UHP / non-null label

Can't be done.  The only egress label knob under `protocols ldp` is `explicit-null`:
```
admin@pe1# set protocols ldp ?
Possible completions:
...
  explicit-null        Advertise the EXPLICIT_NULL label for egress FECs
...
```
`ultimate-hop-popping` under `protocols mpls` is for RSVP, and the IS-IS SR one sets the SR P-flag, so neither helps.  A Junos LDP egress advertises 3 or explicit-null and that's it.

### UHP / Explicit Null

```
set protocols ldp explicit-null
```
PE2 advertises label 2:
```
admin@pe2> show ldp database 
Input label database, 203.0.113.103:0--203.0.113.102:0
Labels received: 3
  Label     Prefix
     16      2001:db8:10::1/128
      2      2001:db8:10::2/128
     17      2001:db8:10::3/128

Output label database, 203.0.113.103:0--203.0.113.102:0
Labels advertised: 3
  Label     Prefix
     17      2001:db8:10::1/128
     16      2001:db8:10::2/128
      2      2001:db8:10::3/128
```
P swaps 17 for 2:
```
admin@p> show route table mpls.0 label 17 

mpls.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[LDP/9] 00:03:22, metric 1
                    >  to fe80::aac1:abff:fecf:4ef5 via ge-0/0/1.0, Swap 2
```
And on PE2 label 2 means "look it up in inet6.0":
```
admin@pe2> show route table mpls.0 

mpls.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 00:02:31, metric 1
                       to table inet.0
0(S=0)             *[MPLS/0] 00:02:31, metric 1
                       to table mpls.0
1                  *[MPLS/0] 00:02:31, metric 1
                       Receive
2                  *[MPLS/0] 00:02:31, metric 1
                       to table inet6.0
2(S=0)             *[MPLS/0] 00:02:31, metric 1
                       to table mpls.0
13                 *[MPLS/0] 00:02:31, metric 1
                       Receive
16                 *[LDP/9] 00:00:41, metric 1
                    >  to fe80::aac1:abff:fea2:2b74 via ge-0/0/1.0, Swap 2
17                 *[LDP/9] 00:00:34, metric 1
                    >  to fe80::aac1:abff:fea2:2b74 via ge-0/0/1.0, Swap 16
```
IPv6 works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1001ms
```
That was tested before `family inet` was added.  I didn't re-test explicit null with it in this lab, but in the SR lab it made no difference, label 2 is IPv6 only on Junos.

#### On the wire

Frames 5 and 6 are IPv4 echo requests under label 2 with S=1, no replies ([ldpjunos_explicit_null_p_to_pe2.pcap](pcaps/ldpjunos_explicit_null_p_to_pe2.pcap)):
```
1	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	16	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	16	2001:db8:187::3	2001:db8:187::2	ICMPv6
5	0x8847	2	203.0.113.1	203.0.113.2	ICMP
6	0x8847	2	203.0.113.1	203.0.113.2	ICMP
```

## BGP labelled-unicast

Same as the SR lab: Junos has no `extended-nexthop` for `family inet labeled-unicast`, so it needs a `local-ipv4-address` and an import policy per remote PE to rewrite the next-hop to the IPv6 loopback:
```
delete protocols bgp group IBGP family inet unicast
set protocols bgp group IBGP family inet labeled-unicast local-ipv4-address 198.51.100.1
set policy-options policy-statement LU-NH-V6 term pe2 from neighbor 2001:db8:10::3
set policy-options policy-statement LU-NH-V6 term pe2 then next-hop 2001:db8:10::3
set protocols bgp group IBGP family inet labeled-unicast import LU-NH-V6
```
Then IPv4 works with the BGP label (18) under the LDP label (17):
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:00:04, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fec3:1aad via ge-0/0/1.0, Push 18, Push 17(top)
```
One difference from SR: with LDP `explicit-null`, P swaps 17 for 2 even when there's a BGP label underneath, so PE2 gets `[2, 18]`.  That's fine, label 2 with S=0 just means "pop and look at the next label".  It's the stack shown in 4PE draft section 8.2 ([bgplu_ldpjunos_p_to_pe2_v4.pcap](pcaps/bgplu_ldpjunos_p_to_pe2_v4.pcap)):
```
1	0x8847	2,18	203.0.113.1	203.0.113.2	ICMP
2	0x8847	16,18	203.0.113.2	203.0.113.1	ICMP
3	0x8847	2,18	203.0.113.1	203.0.113.2	ICMP
4	0x8847	16,18	203.0.113.2	203.0.113.1	ICMP
```
Same downsides as SR: per-PE policy, an IPv4 address needed on every PE, per-prefix labels, and label 0 only for connected routes.  More in [BGP_LU_NOTES.md](BGP_LU_NOTES.md).

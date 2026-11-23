# ldp-mpls-v6-junos: penultimate hop options

Results of testing the label options for the last hop (P to PE2) in the vjunos-router LDP lab.  Image is vJunos-router 25.4R1-S2.3.  LDP runs over IPv6 transport with `family inet6` only, so the only FECs are the IPv6 loopbacks.

| Mode | PE config (`protocols ldp`) | Label PE2 advertises for 2001:db8:10::3 | P label op | IPv6 ping | IPv4 ping |
|---|---|---|---|---|---|
| UHP / Explicit Null | `explicit-null` | 2 | Swap 2 | OK | Fail (dropped at PE2) |
| UHP / non-null label | not possible | - | - | - | - |
| PHP / Implicit Null | (default) | 3 | Pop | OK | Fail (dropped at P) |

**UHP with a non-null label can't be configured for LDP on Junos.**  The only egress label knob under `protocols ldp` is `explicit-null`:
```
admin@pe1# set protocols ldp ?
Possible completions:
...
  explicit-null        Advertise the EXPLICIT_NULL label for egress FECs
...
```
The `ultimate-hop-popping` statement under `protocols mpls` is for RSVP LSPs only, and IS-IS SR's `ultimate-hop-popping` sets the SR P-flag, so neither affects LDP.  When Junos is the egress for its own loopback FEC, it advertises either 3 or explicit-null.

The config change was made on both PEs in each case.  P also has `explicit-null` in its startup config, but that only affects the label for P's own loopback, so it was left alone.

Captures are in [pcaps](pcaps), taken on the P router's tap2 interface (facing PE2).  The capture filter only matches ICMP / ICMPv6 echo, labelled or not.

## UHP / Explicit Null

This is the lab's startup config:
```
set protocols ldp explicit-null
```
PE2 advertises label 2 for its own loopback:
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
PE1 pushes P's label 17 for both the IPv4 and IPv6 routes from CE2:
```
admin@pe1> show ldp database 
Input label database, 203.0.113.101:0--203.0.113.102:0
Labels received: 3
  Label     Prefix
     16      2001:db8:10::1/128
      2      2001:db8:10::2/128
     17      2001:db8:10::3/128

Output label database, 203.0.113.101:0--203.0.113.102:0
Labels advertised: 3
  Label     Prefix
      2      2001:db8:10::1/128
     16      2001:db8:10::2/128
     17      2001:db8:10::3/128
```
```
admin@pe1> show route 2001:db8:10::3 table inet6.3 

inet6.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:10::3/128 *[LDP/9] 00:00:49, metric 1
                    >  to fe80::aac1:abff:fe76:2db1 via ge-0/0/1.0, Push 17
```
```
admin@pe1> show route 203.0.113.2 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 00:00:29, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe76:2db1 via ge-0/0/1.0, Push 17
```
```
admin@pe1> show route 2001:db8:187::3 table inet6.0 

inet6.0: 14 destinations, 15 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:187::3/128*[BGP/170] 00:00:29, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fe76:2db1 via ge-0/0/1.0, Push 17
```
P swaps 17 for 2 towards PE2:
```
admin@p> show route table mpls.0 

mpls.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0                  *[MPLS/0] 00:11:24, metric 1
                       to table inet.0
0(S=0)             *[MPLS/0] 00:11:24, metric 1
                       to table mpls.0
1                  *[MPLS/0] 00:11:24, metric 1
                       Receive
2                  *[MPLS/0] 00:11:24, metric 1
                       to table inet6.0
2(S=0)             *[MPLS/0] 00:11:24, metric 1
                       to table mpls.0
13                 *[MPLS/0] 00:11:24, metric 1
                       Receive
16                 *[LDP/9] 00:05:34, metric 1
                    >  to fe80::aac1:abff:fe8a:7873 via ge-0/0/0.0, Swap 2
17                 *[LDP/9] 00:00:52, metric 1
                    >  to fe80::aac1:abff:fe0a:f5ce via ge-0/0/1.0, Swap 2
```
PE2 maps label 2 to inet6.0:
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
IPv6 ping works, IPv4 fails:
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=73.0 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=2.98 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.983/38.001/73.020/35.018 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1001ms
```
On the wire from P to PE2 ([ldpjunos_explicit_null_p_to_pe2.pcap](pcaps/ldpjunos_explicit_null_p_to_pe2.pcap)) both IPv6 and IPv4 are sent with label 2:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:a2:2b:74, Dst: aa:c1:ab:0a:f5:ce
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6

Frame 2: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:0a:f5:ce, Dst: aa:c1:ab:a2:2b:74
MultiProtocol Label Switching Header, Label: 16, Exp: 0, S: 1, TTL: 63
    0000 0000 0000 0001 0000 .... .... .... = MPLS Label: 16 (0x00010)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1111 = MPLS TTL: 63
Internet Protocol Version 6, Src: 2001:db8:187::3, Dst: 2001:db8:187::2
Internet Control Message Protocol v6

Frame 5: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:a2:2b:74, Dst: aa:c1:ab:0a:f5:ce
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
    0000 0000 0000 0000 0010 .... .... .... = MPLS Label: IPv6 Explicit-Null (2)
    .... .... .... .... .... 000. .... .... = MPLS Experimental Bits: 0
    .... .... .... .... .... ...1 .... .... = MPLS Bottom Of Label Stack: 1
    .... .... .... .... .... .... 0011 1110 = MPLS TTL: 62
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
This is the same as the SR lab: the IPv4 packet reaches PE2 with label 2, is looked up in inet6.0, and is dropped.

## PHP / Implicit Null

Config change on both PEs:
```
delete protocols ldp explicit-null
```
PE2 now advertises label 3 (implicit null) for its loopback:
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
PE1 still pushes 17:
```
admin@pe1> show route 2001:db8:10::3 table inet6.3 

inet6.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:10::3/128 *[LDP/9] 00:00:05, metric 1
                    >  to fe80::aac1:abff:fe76:2db1 via ge-0/0/1.0, Push 17
```
P pops label 17:
```
admin@p> show route table mpls.0 label 17 exact 

mpls.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

17                 *[LDP/9] 00:00:08, metric 1
                    >  to fe80::aac1:abff:fe0a:f5ce via ge-0/0/1.0, Pop
```
```
admin@p> show route table mpls.0 label 17 exact extensive | match "Next hop|operation" 
                Next hop type: Router, Next hop index: 609
                Next hop: fe80::aac1:abff:fe0a:f5ce via ge-0/0/1.0, selected
                Label operation: Pop
```
As with SR, the PFE next-hop for the pop is `MPLS->IPv6`:
```
admin@p> request pfe execute target fpc0 command "show nhdb id 609" 
SENT: Ukern command: show nhdb id 609

   ID      Type      Interface    Next Hop Addr    Protocol       Encap     MTU               Flags  PFE internal Flags
-----  --------  -------------  ---------------  ----------  ------------  ----  ------------------  ------------------
  609   Unicast  ge-0/0/1.0     -                MPLS->IPv6      Ethernet  9178  0x0000000000200001  0x0000000000000002
```
IPv6 ping works, IPv4 fails.  P's PFE "Normal discard" counter goes up by 2 during the 2 packet IPv4 ping:
```
admin@p> show pfe statistics traffic | match "Normal discard" 
    Normal discard             :                   23
```
```
root@ce1:~# ping -c 2 -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=15.1 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=2.89 ms

--- 2001:db8:187::3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.894/9.001/15.109/6.107 ms
```
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1023ms
```
```
admin@p> show pfe statistics traffic | match "Normal discard" 
    Normal discard             :                   25
```
From P to PE2 ([ldpjunos_implicit_null_p_to_pe2.pcap](pcaps/ldpjunos_implicit_null_p_to_pe2.pcap)) only the IPv6 echo request and reply are seen.  The requests are unlabelled IPv6 (118 bytes) and the replies from PE2 carry a label (122 bytes):
```
    1   0.000000 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x0016, seq=1, hop limit=62
    2   0.009655 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x0016, seq=1, hop limit=63 (request in 1)
    3   0.997390 2001:db8:187::2 → 2001:db8:187::3 ICMPv6 118 Echo (ping) request id=0x0016, seq=2, hop limit=62
    4   0.998105 2001:db8:187::3 → 2001:db8:187::2 ICMPv6 122 Echo (ping) reply id=0x0016, seq=2, hop limit=63 (request in 3)
```

## Summary

LDP gives exactly the same results as SR-MPLS on Junos.  The forwarding behaviour depends on the LSP being for an IPv6 FEC, not on how the label was signalled:

* **Explicit null**: IPv4 arrives at PE2 with label 2, which is looked up in inet6.0, and is dropped.
* **Implicit null**: P's pop next-hop is `MPLS->IPv6`, so the IPv4 packet is discarded in P's PFE.
* **Non-null label**: Junos LDP has no option to advertise a non-null label for a local FEC, so this mode can't be tested.  Based on the SR result, a Junos egress would most likely map that label to inet6.0 as well.

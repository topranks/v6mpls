# sr-mpls-v6-xrd

IPv6 only MPLS core lab based on Cisco xRD-control-plane, which simulates IOS-XR platforms.  The two CE devices are based on vanilla debian container image and run FRR.

Don't forget you need these sysctl's for XRd:
```
sysctl -w fs.inotify.max_user_instances=64000
sysctl -w fs.inotify.max_user_watches=64000
```

## TL;DR

| Mode | PE config (`router isis IGP / interface Loopback0 / address-family ipv6 unicast`) | P label op for 16003 | IPv6 | IPv4 |
|---|---|---|---|---|
| PHP / Implicit Null (lab default) | `prefix-sid index N` | Pop | OK | Fail |
| UHP / non-null label | `prefix-sid index N php-disable` | Swap 16003 | OK | Fail |
| UHP / Explicit Null | `prefix-sid index N explicit-null` | Swap 2 | OK | Fail |

N is 1 on PE1 and 3 on PE2.  IPv4 fails the same way in every mode: PE1 never puts a usable IPv4 packet on the LSP, so the last hop never gets a say.  This is the control plane image, real XR hardware may well behave differently.

Captures referenced below are in [pcaps](pcaps), taken in P's netns on Gi0-0-0-0 (facing PE1) or Gi0-0-0-1 (facing PE2).  Output is summarised with:
```
tshark -r <file> -T fields -e frame.number -e eth.type -e mpls.label -e _ws.col.Source -e _ws.col.Destination -e _ws.col.Protocol
```

## Tests

### Ingress PE

Same in all modes.  PE1 has the IPv4 route from CE2 recursive via PE2's IPv6 loopback, with the "next hop VRF" pointing at the default IPv6 table (0xe0800000).  Note there is no label anywhere in it:
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
Whereas the IPv6 loopback it recurses through does have one:
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

    Load distribution: 0 (refcount 3)

    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/1    fe80::a8c1:abff:fe93:753c
```
With the lab as-is, an IPv4 ping sends nothing at all towards P ([srxrd_implicit_null_v4_pe1_to_p.pcap](pcaps/srxrd_implicit_null_v4_pe1_to_p.pcap) is empty):
```
root@ce1:~# ping -c 2 -I 203.0.113.1 203.0.113.2
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1023ms
```

#### What about IPv4 on the core interface?

Adding `family inet` fixed things on Junos, so I tried the XR equivalent on the PEs, with an IPv4 loopback (not advertised anywhere) and unnumbered on the core interface.  P still has no IPv4:
```
interface Loopback0
 ipv4 address 192.0.2.1 255.255.255.255
interface GigabitEthernet0/0/0/1
 ipv4 point-to-point
 ipv4 unnumbered Loopback0
```
(XR won't take `ipv4 unnumbered` on Ethernet without `ipv4 point-to-point`.)

Now PE1 does send the IPv4 packets towards P:
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
But they go with the MPLS Ethertype and **no label** ([unnum_srxrd_expnull_pe1_to_p.pcap](pcaps/unnum_srxrd_expnull_pe1_to_p.pcap)).  tshark tries to read the IPv4 header as a label stack, hence the garbage labels.  Frame 6 is a gratuitous ARP for 192.0.2.1 when IPv4 came up on the interface:
```
1	0x8847	282624,375492,258070	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	MPLS
2	0x8847	282624,378436,258070	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	MPLS
3	0x8847	282624,381972,258070,831495	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	MPLS
4	0x8847	282624,385108,258070	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	MPLS
5	0x8847	282624,385860,258070	aa:c1:ab:aa:3c:80	aa:c1:ab:93:75:3c	MPLS
6	0x0806		aa:c1:ab:aa:3c:80	ff:ff:ff:ff:ff:ff	ARP
```
The hex makes it obvious, `88 47` then straight into `45 00`, the start of the IPv4 header:
```
0000  aa c1 ab 93 75 3c aa c1 ab aa 3c 80 88 47 45 00   ....u<....<..GE.
0010  00 54 5b ac 40 00 3f 01 67 f8 cb 00 71 01 cb 00   .T[.@.?.g...q...
```
P reads 0x45000 (282624) as a label and drops it.  Same in all three modes, so I put the startup configs back without IPv4.  Detail in [IPV4_UNNUMBERED_NOTES.md](IPV4_UNNUMBERED_NOTES.md).

### PHP / Implicit Null

```
RP/0/RP0/CPU0:pe2#show isis database pe2.00-00 verbose | include Prefix-SID
    Prefix-SID Index: 3, Algorithm:0, R:0 N:1 P:0 E:0 V:0 L:0
```
P pops:
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
IPv6 goes from P to PE2 unlabelled ([srxrd_implicit_null_p_to_pe2.pcap](pcaps/srxrd_implicit_null_p_to_pe2.pcap)):
```
1	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	16001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x86dd		2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	16001	2001:db8:187::3	2001:db8:187::2	ICMPv6
```

### UHP / non-null label

```
router isis IGP
 interface Loopback0
  address-family ipv6 unicast
   prefix-sid index 3 php-disable
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
PE2 installs 16003 as "Aggregate", i.e. pop and look up in a table:
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
IPv6 with 16003 from P to PE2 ([srxrd_uhp_nonnull_p_to_pe2.pcap](pcaps/srxrd_uhp_nonnull_p_to_pe2.pcap)):
```
1	0x8847	16003	2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	16001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x8847	16003	2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	16001	2001:db8:187::3	2001:db8:187::2	ICMPv6
```
Whether PE2 would handle IPv4 under 16003 we can't tell, PE1 never sends any.

### UHP / Explicit Null

```
router isis IGP
 interface Loopback0
  address-family ipv6 unicast
   prefix-sid index 3 explicit-null
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
And PE2 sends label 2 to the IPv6 table (0xe08fffff):
```
RP/0/RP0/CPU0:pe2#show cef mpls local-label 2 eos detail | inc "via-|next hop VRF" 
Sun Nov 22 01:29:55.500 UTC
    recursion-via-/128
    next hop VRF - '', table - 0xe08fffff
```
IPv6 with label 2 from P to PE2, no IPv4 ([srxrd_explicit_null_p_to_pe2.pcap](pcaps/srxrd_explicit_null_p_to_pe2.pcap)):
```
1	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
2	0x8847	16001	2001:db8:187::3	2001:db8:187::2	ICMPv6
3	0x8847	2	2001:db8:187::2	2001:db8:187::3	ICMPv6
4	0x8847	16001	2001:db8:187::3	2001:db8:187::2	ICMPv6
```

## BGP labelled-unicast

Not possible on XRd 24.2.11.  XR happily takes `address-family ipv4 unicast` on an IPv6 neighbor (RFC 8950, how the lab works today), but not `ipv4 labeled-unicast`:
```
!! SEMANTIC ERRORS: This configuration was rejected by 
!! the system due to semantic errors. The individual 
!! errors with each failed configuration command can be 
!! found below.


router bgp 64496
 neighbor-group IBGP
  address-family ipv4 labeled-unicast
!!% Change would result in neighbor (2001:db8:10::3) being activated with an invalid address family
   next-hop-self
!!% 'BGP' detected the 'warning' condition 'Neighbor/group is not activated for this address family'
  !
 !
!
end
```
Under the neighbor directly it doesn't even exist:
```
!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to
!! one or more of the following reasons:
!!  - the entered commands do not exist,
!!  - the entered commands have errors in their syntax,
!!  - the software packages containing the commands are not active,
!!  - the current user is not a member of a task-group that has 
!!    permissions to use the commands.

address-family ipv4 labeled-unicast
```
With no IPv4 between the PEs there's no IPv4 session to run it over instead, so that's the end of that.  RFC 8950 does cover labelled-unicast (AFI 1 / SAFI 4), so it's an XR limitation not a standards one.  More in [BGP_LU_NOTES.md](BGP_LU_NOTES.md).

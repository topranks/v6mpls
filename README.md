# Mixed Messages: IPv4, IPv6 and the labels inbetween

Can an MPLS network built on an IPv6-only core transit IPv4 packets?

This repo contains several [containerlab](https://containerlab.dev/) topologies to test that proposition.

## Background

[Daryll Swer](https://www.swernetworks.com/author/daryll-swer/) nerd-sniped me on this one a few weeks back, drawing my attention to a "bug" in Juniper MX devices whereby IPv4 traffic could not be carried over an MPLS network with an IPv6-only core.  He further said the same setup works just fine with Arista devices.

The question piqued my interest for two reasons:

1. Why exactly doesn't it work, and can it be made to work?
3. What do the standards say about this kind of scenario?  What should we expect to happen?

## Scenario

The exact scenario we want to look at is where we have an MPLS network, composed of PE and P routers, which only have IPv6 addressing on their links, and only an IPv6 loopback address. 
If we connect external IPv4 networks to the PEs, can we use the MPLS LSPs built across the IPv6 core to transport them across the network?  In the containerlab topologies we use the most basic network of this kind that we can, with only three core routers, PE1, P and PE2.

Configuration wise we enable an IGP on link and loopback interfaces, in this lab IS-IS.  We also enable MPLS, and for most of the tests use [SR-MPLS](https://www.rfc-editor.org/info/rfc8660/) with signalling carried in IS-IS itself, but also for completeness test with [LDP](https://www.ietf.org/proceedings/85/rfc/rfc5036.txt) in one lab.

With that in place every PE learns a route for the IPv6 loopback of every other PE through IS-IS, and an MPLS LSP is built to use for traffic to that IP.

We then enable IBGP between the PE's IPv6 loopback addresses.  We enable both IPv6 and IPv4 address families on the IBGP sessions, and configure the PEs to use the methods described in [RFC 8950]https://www.rfc-editor.org/info/rfc8950/) to advertise IPv4 addresses with a next-hop of their IPv6 loopback.

Lastly we connect two CE devices to the network.  One is connected to PE1, the other to PE2.  The CE devices have two EBGP sessions each to the PEs.  One over the IPv6 addressing on the link from PE to CE, and another over the IPv4 addressing from PE to CE.  This IPv4 link addressing facing the CEs is the only IPv4 addressing on the "core" network.

[![v6mpls lab topology](diagram/v6mpls_lab.png)](diagram/v6mpls_lab.pdf?raw=true)

As a result of this any IPv4 route that CE1 announces to PE1 gets propagated to PE2 in IBGP, with the next-hop set to the loopback IPv6 address of PE1.  The question we want to answer is what will PE2 on receipt of this route?  Given that the next-hop is an IPv6 address - _reachable via an MPLS path_ - will the route be installed?  And how will the data-plane encapsulation work on the wire for traffic to this destination?

## Protocol Notes

### Penultimate Hop Popping

[Penultimate Hop Popping](https://www.rfc-editor.org/info/rfc3031/#section-3.16) is in the original MPLS standards and defines the label operations the last 'P' router in an LSP performs.

The most basic form ("implicit null") involves the last P router **popping** the last label off the packet, leaving just the raw IPv4 or IPv6 (or other) payload.  The advantage of transmitting the packet from last P to last PE router without an MPLS label means that egress PE device only sees an IP header, which it uses to forward the packet.  If a label is present between P and PE then the PE has to first examine the label, and then look at the IP header which is a more expensive operation.

Another form of PHP is also defined, called "explicit null" (see [RFC 3032](https://www.rfc-editor.org/rfc/rfc3032.html#section-2)).  In this case the last P router performs a label **swap** operation, applying a well-known label to the packet which indicates the payload protocol.  Specifically the explicit null label can be:

0 - Indicating the payload is an IPv4 packet

2 - Indicating the payload is an IPv6 packet

### MPLS Label Operations

It is easy to forget today that MPLS was developed at a time when doing longest-prefix matching in hardware was considered by some to be an impossibility, and when telco's were still keen on technologies such as [ATM](https://en.wikipedia.org/wiki/Asynchronous_Transfer_Mode), which switched "cells" by doing exact-matches on fixed-length headers.  [RFC 3031](https://www.rfc-editor.org/info/rfc3031/#section-2.1) alludes to this stating _"MPLS forwarding can be done by switches which are capable of doing label lookup and replacement, but are either not capable of analyzing the network layer headers, or are not capable of analyzing the network layer headers at adequate speed."_

TCAMs that could perform LPM in hardware had become available by the time MPLS began to be widely deployed, so it was other properties of MPLS that led to its adoption.  But it's worth remembering this background when considering how it works, and observing how the routers in our virtual labs behave.  Two more statements from RFC 3031 which will become relevant are:

_"In MPLS, the assignment of a particular packet to a particular FEC is done just once, as the packet enters the network."_

_"In the MPLS forwarding paradigm, once a packet is assigned to a FEC, no further header analysis is done by subsequent routers; all forwarding is driven by the labels."_



## Labs

### JunOS vjunos-router

The first vendor we examine is Juniper, using their vjunos-router virtual device.  With the lab spun up we can see that we learn an IPv6 and IPv4 route on PE2 from CE2:
```
admin@pe2> show route receive-protocol bgp 198.51.100.6 table inet.0 

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
  Prefix		  Nexthop	       MED     Lclpref    AS path
* 203.0.113.2/32          198.51.100.6         0                  64498 ?
```
```
admin@pe2> show route receive-protocol bgp 2001:db8:50::5 table inet6.0                         

inet6.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
  Prefix		  Nexthop	       MED     Lclpref    AS path
* 2001:db8:187::3/128     2001:db8:50::5       0                  64498 ?
```

On PE1 we can see both of these routes are learnt in IBGP, and in both cases the BGP route's next-hop is the IPv6 loopback of PE1:
```
admin@pe1> show route table inet6.0 2001:db8:187::3/128 extensive | match "protocol next hop"        
                Protocol next hop: 2001:db8:10::3
```
```
admin@pe1> show route table inet.0 203.0.113.2 extensive | match "protocol next hop"    
                Protocol next hop: 2001:db8:10::3
```
If we check the route to this next-hop we see an entry in inet6.3, which means MPLS will be used to carry it:
```
admin@pe1> show route 2001:db8:10::3 table inet6.3 

inet6.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:10::3/128 *[L-ISIS/14] 03:23:25, metric 201
                    >  to fe80::aac1:abff:fedc:446f via ge-0/0/1.0, Push 400003
```
If we look at routes originated by CE2 again we see that PE1 will use the same label to get to those, based on the same next-hop:
```
admin@pe1> show route table inet6.0 2001:db8:187::3/128                                               

inet6.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:187::3/128*[BGP/170] 03:21:56, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fedc:446f via ge-0/0/1.0, Push 400003
```
```
admin@pe1> show route table inet.0 203.0.113.2                                                        

inet.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

203.0.113.2/32     *[BGP/170] 03:22:02, MED 0, localpref 100, from 2001:db8:10::3
                      AS path: 64498 ?, validation-state: unverified
                    >  to fe80::aac1:abff:fedc:446f via ge-0/0/1.0, Push 400003
```

On our P router we can see the label operations it will apply for packets with this label:
```
admin@p> show route table mpls.0 label 400003 exact 

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 03:34:55, metric 101
                    >  to fe80::aac1:abff:feaa:c085 via ge-0/0/1.0, Swap 2
```

We can see that the router is configured here to do explicit null PHP.  It is going to swap the incoming label of 40003 for the IPv6 explicit-null label of 2 and forward it out ge-0/0/1 to PE2.  But as we seen on PE1, 400003 will be used for the IPv6 and IPv4 destination address, might using label '2' cause a problem with IPv4?

#### On the wire

Let's try to ping from CE1 to CE2 over their IPv6 loopbacks:
```
root@ce1:~# ping -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=2.96 ms
```
If we do a capture on the link from P to PE2 we can see how the echo request packet is encapsulated:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:1c:a4:04 (aa:c1:ab:1c:a4:04), Dst: aa:c1:ab:aa:c0:85 (aa:c1:ab:aa:c0:85)
    Type: MPLS label switched packet (0x8847)
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6

```

If we try to ping from CE1 to CE2 over the IPv4 loopbacks it fails, however:
```
root@ce1:~# ping -I 203.0.113.1 -c 2 203.0.113.2 
PING 203.0.113.2 (203.0.113.2) from 203.0.113.1 : 56(84) bytes of data.

--- 203.0.113.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1029ms
```
Looking on the wire from P to PE2 we can see the frame is transmitted to PE2:
```
Frame 1: 102 bytes on wire (816 bits), 102 bytes captured (816 bits)
Ethernet II, Src: aa:c1:ab:1c:a4:04 (aa:c1:ab:1c:a4:04), Dst: aa:c1:ab:aa:c0:85 (aa:c1:ab:aa:c0:85)
    Type: MPLS label switched packet (0x8847)
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
Internet Protocol Version 4, Src: 203.0.113.1, Dst: 203.0.113.2
Internet Control Message Protocol
```
But interestingly with tcpdump it shows up as invalid, tcpdump does not think there should be an IPv4 packet inside an MPLS label of '2':
```
18:15:20.803764 aa:c1:ab:1c:a4:04 > aa:c1:ab:aa:c0:85, ethertype MPLS unicast (0x8847), length 102: MPLS (label 2 (IPv6 explicit NULL), tc 0, [S], ttl 62)
	 [version 4 != 6] (invalid)
```
What is happening when the packet reaches PE2 then?  If we look at the MPLS forwarding table it becomes clear enough:
```
admin@pe2> show route table mpls.0 label 2 exact 

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2                  *[MPLS/0] 03:50:54, metric 1
                       to table inet6.0
```

This label operation is hard-coded in JunOS.  Any packet arriving with an MPLS label of 2 is looked up in the inet6 forwarding table.  Which works fine if the source is an IPv6 packet, but obviously looking up an IPv4 destination in the IPv6 routing table will fail, so the packet just gets dropped.

#### What about implicit-null?

What if we tell the router not to use label 2 across the P->PE2 link?  And instead pop the label PE1 added and just forward the remaining IP packet to PE2?  On PE2 we tell it to not signal to use explicit-null:
```
delete protocols isis source-packet-routing explicit-null
```
After which we see the MPLS entry on P has changed, it will now POP the incoming label rather than swap for 2:
```
admin@p> show route table mpls.0 label 400003 exact    

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

400003             *[L-ISIS/14] 00:00:26, metric 101
                    >  to fe80::aac1:abff:feaa:c085 via ge-0/0/1.0, Pop
```
IPv6 ping still works fine:
```
root@ce1:~# ping -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=2.69 ms
```
We see on the link from P to PE the packet is sent as raw IPv6:
```
Frame 1: 118 bytes on wire (944 bits), 118 bytes captured (944 bits)
Ethernet II, Src: aa:c1:ab:1c:a4:04 (aa:c1:ab:1c:a4:04), Dst: aa:c1:ab:aa:c0:85 (aa:c1:ab:aa:c0:85)
    Type: IPv6 (0x86dd)
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```
However when we try an IPv4 ping it fails.  And we see nothing on the wire between P and PE2.  Looking closer at the new MPLS forwarding entry we see some interesting things:
```
admin@p> show route table mpls.0 label 400003 exact extensive         

mpls.0: 14 destinations, 14 routes (14 active, 0 holddown, 0 hidden)
400003 (1 entry, 1 announced)
        *L-ISIS Preference: 14
                Next hop type: Router, Next hop index: 607
                Next hop: fe80::aac1:abff:feaa:c085 via ge-0/0/1.0, selected
                Label operation: Pop      
```
We can see that next-hop index in the PFE is set up to use Ethernet encap for IPv6:
```
VMX-0(p vty)# show nhdb id 607 extensive
   ID      Type      Interface    Next Hop Addr    Protocol       Encap     MTU               Flags  PFE internal Flags
-----  --------  -------------  ---------------  ----------  ------------  ----  ------------------  ------------------
  607   Unicast  ge-0/0/1.0     -                MPLS->IPv6      Ethernet  9178  0x0000000000200001  0x0000000000000002
```
In theory that would mean the P router tries to place the IPv4 payload in an Ethernet frame with Ethertype 0x86dd (IPv6).  This is an invalid packet, so I'm guessing the P ends up dropping it somewhere internally instead.

But with either explicit or implicit null enabled the results are clear: Juniper applies a fixed set of label and encap operations for a packet using this LSP.  Because the prefix that created the LSP is an IPv6 address those operations match what is needed to carry IPv6.  If we try to send IPv4 packets over this LSP they don't get to the destination, either being dropped at PE2 when using explicit null (because label=2 directs it to use inet6.0), or at the P router when using implicit null (which can't put an IPv4 packet inside an Ethernet frame with IPv6 Ethertype).

### Cisco xRD control-plane

Next we build the same topology using Cisco xRD control-plane containers, which simulate their IOS-XR platform.  Agin the CEs use FRR in a plain Linux container, PE2 learns the same routes as before from CE2 for example:
```
RP/0/RP0/CPU0:pe2#show bgp ipv6 unicast neighbor 2001:db8:50::5 routes | begin "^   Network"
Sun Nov 22 00:56:04.363 UTC
   Network            Next Hop            Metric LocPrf Weight Path
*> 2001:db8:187::3/128
                      2001:db8:50::5           0             0 64498 ? N
```
```
RP/0/RP0/CPU0:pe2#show bgp ipv4 unicast neighbor 198.51.100.6 routes | begin "^   Network"
Sun Nov 22 00:54:34.228 UTC
   Network            Next Hop            Metric LocPrf Weight Path
*> 203.0.113.2/32     198.51.100.6             0             0 64498 ? N
```

On PE1 both routes have PE2's IPv6 loopback as BGP next-hop, same as on Juniper:
```
RP/0/RP0/CPU0:pe1#show bgp ipv6 unicast 2001:db8:187::3/128 | begin "^  6"  
Sun Nov 22 00:59:13.739 UTC
  64498
    2001:db8:10::3 (metric 201) from 2001:db8:10::3 (203.0.113.103)
      Origin incomplete, metric 0, localpref 100, valid, internal, best, group-best
      Received Path ID 0, Local Path ID 1, version 7
```
```
RP/0/RP0/CPU0:pe1#show bgp ipv4 unicast 203.0.113.2/32 | begin "^  6"
Sun Nov 22 00:59:38.270 UTC
  64498, (received & used)
    2001:db8:10::3 (metric 201) from 2001:db8:10::3 (203.0.113.103)
      Origin incomplete, metric 0, localpref 100, valid, internal, best, group-best
      Received Path ID 0, Local Path ID 1, version 6
```

Checking the route to the PE2 loopback on PE1 we can see: it is known from IS-IS with a SR-MPLS label:
```
RP/0/RP0/CPU0:pe1#show route ipv6 2001:db8:10::3/128

Routing entry for 2001:db8:10::3/128
  Known via "isis IGP", distance 115, metric 201, labeled SR
  Local Label 16003, type level-2
  Installed Nov 22 00:32:14.962 for 00:02:59
  Routing Descriptor Blocks
    fe80::a8c1:abff:fede:ff15, from 2001:db8:10::3, via GigabitEthernet0/0/0/1
```
Looking at the IPv6 CEF route we see it is "via" the same PE2 loopback IP, and that label 16003 will be used:
```
RP/0/RP0/CPU0:pe1#show cef ipv6 2001:db8:187::3/128 | beg "   via"
Sun Nov 22 01:04:52.933 UTC
   via 2001:db8:10::3/128, 3 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0x8724e7a8 0x0]
    next hop 2001:db8:10::3/128 via 2001:db8:10::3/128
```
```
RP/0/RP0/CPU0:pe1#show cef ipv6 2001:db8:187::3/128 internal | inc LABEL_INFO
Sun Nov 22 01:09:26.882 UTC
          LABEL_INFO => [default [o-label:16003 l-label:16003 type:0 flags:[linked to ipv4, linked to lsw, ipv6, recycle] ext-flags:[none] stats-key:[0x3e8300000000]]] obj-id:[N/A][0x3089344818]]}
              LABEL_INFO => [default [o-label:16003 l-label:16003 type:0 flags:[linked to ipv4, linked to lsw, ipv6, recycle] ext-flags:[none] stats-key:[0x3e8300000000]]] obj-id:[N/A][0x3089344818]]}
```
This is true for the IPv4 route, but note the "next-hop VRF" line.  This indicates the next-hop is known in another VRF/routing instance.  In this case 0xe0800000 corresponds to the default IPv6 table rather than any named VRF.
```
RP/0/RP0/CPU0:pe1#show cef ipv4 203.0.113.2/32 | beg "   via"
Sun Nov 22 01:06:17.905 UTC
   via 2001:db8:10::3/128, 3 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0x8724e7a8 0x0]
    next hop VRF - 'default', table - 0xe0800000
    next hop 2001:db8:10::3/128 via 2001:db8:10::3/128
```
```
RP/0/RP0/CPU0:pe1#show cef ipv4 203.0.113.2/32 internal | inc LABEL_INFO     
Sun Nov 22 01:10:06.640 UTC
          LABEL_INFO => [default [o-label:16003 l-label:16003 type:0 flags:[linked to ipv4, linked to lsw, ipv6, recycle] ext-flags:[none] stats-key:[0x3e8300000000]]] obj-id:[N/A][0x3089344818]]}
              LABEL_INFO => [default [o-label:16003 l-label:16003 type:0 flags:[linked to ipv4, linked to lsw, ipv6, recycle] ext-flags:[none] stats-key:[0x3e8300000000]]] obj-id:[N/A][0x3089344818]]}
```

So we expect PE1 to impose label of 16003 and transmit the frames to the 'P' router.  If we ping with IPv6 it works:
```
root@ce1:~# ping -I 2001:db8:187::2 2001:db8:187::3
PING 2001:db8:187::3(2001:db8:187::3) from 2001:db8:187::2 : 56 data bytes
64 bytes from 2001:db8:187::3: icmp_seq=1 ttl=61 time=1.70 ms
64 bytes from 2001:db8:187::3: icmp_seq=2 ttl=61 time=1.54 ms
```
And we see PE1 transmitting the packet to the P router with label 16003:
```
Frame 1: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:fa:a7:16 (aa:c1:ab:fa:a7:16), Dst: aa:c1:ab:de:ff:15 (aa:c1:ab:de:ff:15)
    Type: MPLS label switched packet (0x8847)
MultiProtocol Label Switching Header, Label: 16003, Exp: 0, S: 1, TTL: 63
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```

If we look on the 'P' router we see it has a static label swap operation set up for 16003 incoming, changing the label to '2' representing the IPv6 explicit-null.

```
RP/0/RP0/CPU0:p#show mpls forwarding labels 16003 detail
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16003  Exp-Null-v6 SR Pfx (idx 3)     Gi0/0/0/1    fe80::a8c1:abff:fe5a:e1a5   \
                                                                   1834        
     Updated: Nov 22 00:32:18.520
     Version: 14, Priority: 1
     Label Stack (Top -> Bottom): { 2 }
     NHID: 0x20001, Encap-ID: N/A, Path idx: 0, Backup path idx: 0, Weight: 0
     MAC/Encaps: 14/18, MTU: 1500
     Outgoing Interface: GigabitEthernet0/0/0/1 (ifhandle 0x00000010)
     Packets Switched: 20
```

This looks as expected on the wire:
```
Frame 3: 122 bytes on wire (976 bits), 122 bytes captured (976 bits)
Ethernet II, Src: aa:c1:ab:ba:f3:33 (aa:c1:ab:ba:f3:33), Dst: aa:c1:ab:5a:e1:a5 (aa:c1:ab:5a:e1:a5)
    Type: MPLS label switched packet (0x8847)
MultiProtocol Label Switching Header, Label: 2 (IPv6 Explicit-Null), Exp: 0, S: 1, TTL: 62
Internet Protocol Version 6, Src: 2001:db8:187::2, Dst: 2001:db8:187::3
Internet Control Message Protocol v6
```

And on PE2 examining what it does if packet arrives with a single label of '2' we see it references the same "next hop VRF" we seen for the IPv4 route on PE1, the global IPv6 table:
```
RP/0/RP0/CPU0:pe2#show cef mpls local-label 2 eos detail | inc "via-|next hop VRF" 
Sun Nov 22 01:29:55.500 UTC
    recursion-via-/128
    next hop VRF - '', table - 0xe08fffff
```

So all good for IPv6.  What happens to our IPv4 pings though?  NOTHING.

PE1 increments the counters for packets switched with label 16003 when we start our ping.  But a tcpdump on the wire reveals no MPLS packet is sent to P.  I ran some debugs and there are some esoteric logs but it's hard to pin down exactly what happens.  Suffice to say that while the control plane makes it look like everything is set up and it should forward the packet with label 16003, the actual data-plane code trips up when the payload is IPv4.

Implicit or explicit null at the last P router are irrelevant here so I didn't test implicit.  The first PE fails to send the packet into the LSP which will happen the same in either case.

### Arista cEOS

### JunOS vjunos-router - LDP


## Standards

So is it standards compliant?  No.

Linux?  

Standards don't mention it - but they probably couldn't imagine it.  Time of writing MPLS nobody expected todays scenario - that people would want to try to send IPv4 over an IPv6-only network.  The idea was we would migrate from IPv4 to IPv6 in a few years, and for those few years we would dual-stack everything.  It is only today, with no expiry date for IPv4 on the horizon, that people are trying to do wonky things like v4-over-v6.

## Solutions

### 4PE

4PE is conceptually the "opposite of 6PE ([RFC4798](https://www.rfc-editor.org/info/rfc4798/))".  An internet draft, [draft-mishra-idr-v4-islands-v6-core-4pe-10](https://datatracker.ietf.org/doc/html/draft-mishra-idr-v4-islands-v6-core-4pe-10)), titled "Connecting IPv4 Islands over IPv6 Core using IPv4 Provider Edge Routers (4PE)" was published several years back.

Section 4 of the draft standards acknowledges the problem with implicit null at last P.  It doesn't refer to Ethertype.  "With the Default implicit-null Penultimate Hop (PHP) mode, the egress LSR P node would POP the topmost label revealing the native IPv4 packet which would be subsequently dropped as the Core underlay is an IPv6-Only core."  <--- I think this gets it wrong though, it doesn't POP + new routing lookup, it's label lookup -> pop+encap rule.  The packet after popping may be IPv4, but many platforms don't look at it, instead the ingress label points to a and on an Ethernet link to set the correct destination MAC address and Ethertype.


"8.4. Arbitrary topmost with customer prefixes unlabeled" - a random label for the IPv6 loopback is assigned, egress PE advertises it to neighbors, and says it does not want PHP??

"8.5 - Explicit Null topmost with customer prefixes unlabeled" - this is the mode we see the Juniper doing but failing to process.  it means forward it all the way, do explicit null swap on last PE of 16003 -> 2, and then have the egress PE pop the '2', but look at the payload and route dynamically.







MAC known from ND.  Etype known from packet inspection.

Conceptually 



### VPNv4

### Standardise Arista's approach

The cleanest way 

- Ingress PE pushes two labels like in 4PE/VPNv4
- We don't exchange labels with MPLS though, the ingress PE always pushes 0 for an IPv4 packet or 2 for IPv6, with the S-bit set to 1.
- The transport label is pushed on top of this by the ingress PE



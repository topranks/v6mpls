# sr-mpls-v6-xrd

IPv6 only MPLS core lab based on Cisco xRD-control-plane, which simulates IOS-XR platforms.  The two CE devices are based on vanilla debian container image and run FRR.

### Tests / results

With the lab spun up PE2 learns the same routes as seen in the Juniper example, for instance:
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

# sr-mpls-v6-junos

IPv6 only MPLS core lab based on vjunos-router, with two CE devices based on vanilla debian container image.

## vjunos-router

The Juniper vjunos-router images are available free of charge at the below URL:

https://support.juniper.net/support/downloads/?p=vjunos-router

## Tests

With the lab spun up we can see that we learn an IPv6 and IPv4 route on PE2 from CE2:
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


# IPv4 routes with IPv6 next-hops & MPLS

## Background

[Daryll Swer](https://www.swernetworks.com/author/daryll-swer/) nerd-sniped me on this one a few weeks back, drawing my attention to a "bug" in Juniper MX devices whereby IPv4 traffic could not be carried over an MPLS network with an IPv6-only core.  He further said the same setup works just fine with Arista devices.

The question piqued my interest for two reasons:

1. Why exactly doesn't work, and can it be made to work?
3. What do the standards say about this kind of scenario?  What should we expect here?

## Scenario

The exact scenario we want to look at is where we have an MPLS network, composed of PE and P routers, which only have IPv6 addressing on their links, and only an IPv6 loopback address. To test the how such a network behaves when PEs have external IPv4 routes we create the most basic virtual lab we can, consisting of only three routers, PE1, P and PE2.

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

It is easy to forget today that MPLS was developed at a time when doing longest-prefix matching in hardware was considered by some to be an impossibility, and at a time when telco's were still keen on technologies such as ATM, which switched based exact matching of fixed-length virtual-circuit identifiers.  [RFC 3031](https://www.rfc-editor.org/info/rfc3031/#section-2.1) alludes to this stating _"MPLS forwarding can be done by switches which are capable of doing label lookup and replacement, but are either not capable of analyzing the network layer headers, or are not capable of analyzing the network layer headers at adequate speed."_  One of the protocols immediate predecessors, Cisco's tag switching, alludes to how the technology could be adapted to run on the ATM switches of the time, with [RFC2105](https://www.rfc-editor.org/info/rfc2105/#section-5) stating that _"implementing tag switching on an ATM switch would simplify integration of ATM switches and routers - an ATM switch capable of tag switching would appear as a router to an adjacent router."_

TCAMs that could perform LPM in hardware had become available by the time MPLS began to be widely deployed, but MPLS gained wide adoption anyway because of its other properties.  Nonetheless it is worth remembering this background when considering how it works, why it is relevant will become clear when we observe the behaviour of the devices in our virtual labs.  Two more statements from RFC 3031 worth bearing in mind include:

_"In MPLS, the assignment of a particular packet to a particular FEC is done just once, as the packet enters the network."_

_"In the MPLS forwarding paradigm, once a packet is assigned to a FEC, no further header analysis is done by subsequent routers; all forwarding is driven by the labels."_

## Labs

### JunOS vjunos-router

The first vendor we examine is Juniper, using their vjunos-router virtual device.  With the lab spun up


### Cisco xRD control-plane

### Arista cEOS

### JunOS vjunos-router - LDP

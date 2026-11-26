# 4PE: IPv4, IPv6 and the labels inbetween

Can an MPLS network built on an IPv6-only core transit IPv4 packets?  An expired [internet draft](https://datatracker.ietf.org/doc/html/draft-mishra-idr-v4-islands-v6-core-4pe-10) describes several ways to do this with existing protocol options, and indeed there may be more.  But it was never adopted and what vendors support is unclear.

This repo contains several [containerlab](https://containerlab.dev/) topologies to test what works and doesn't across Juniper, Arista and Cisco IOS-XR.

## Background

[Daryll Swer](https://www.swernetworks.com/author/daryll-swer/) nerd-sniped me on this one a few weeks back, drawing my attention to what he described as a "bug" in Juniper MX devices whereby IPv4 traffic could not be carried over an MPLS network with an IPv6-only core.  He explained the same setup works just fine with Arista devices.

The question piqued my interest for two reasons:

1. Why exactly doesn't it work, and can it be made to work?
3. What do the standards say about this setup?  Is it expected to work?

## Scenario

We have an MPLS network, composed of PE and P routers, which only have IPv6 addressing on their links, and only an IPv6 loopback address.

If we connect external IPv4 networks to the PEs, can we use the MPLS LSPs built across the IPv6 core to transport them across the network?  The scenario is the same as that described in the [4PE](https://datatracker.ietf.org/doc/html/draft-mishra-idr-v4-islands-v6-core-4pe-10) RFC draft.

### Nodes

In the containerlab topologies we use the most basic network of this kind possible, with three core routers, PE1, P and PE2, and two CEs, one connected to each PE.

### Protocols

In the lab we use ISIS an IGP protocol, configuring it on link and loopback interfaces.  We also enable MPLS, and for most of the tests use [SR-MPLS](https://www.rfc-editor.org/info/rfc8660/) with signalling carried in [IS-IS](https://www.rfc-editor.org/info/rfc8667/) itself, but also for completeness test with [LDP](https://www.ietf.org/proceedings/85/rfc/rfc5036.txt).

Every PE learns a route for the IPv6 loopback of every other PE through IS-IS, and an MPLS LSP is built to use for traffic to that IP.

We enable IBGP between the PE's IPv6 loopback addresses.  We enable both IPv6 and IPv4 address families on the IBGP sessions, and configure the PEs to use the methods described in [RFC 8950](https://www.rfc-editor.org/info/rfc8950/) to advertise IPv4 addresses with a next-hop of their IPv6 loopback.

The CE devices have two EBGP sessions each to the PEs.  One over the IPv6 addressing on the link from PE to CE, which is used to announce IPv6 addresses, and another over the IPv4 addresses on the same link, used for IPv4 addresses.  The IPv4 addresses on the PE links connecting the CEs are the only IPv4 addresses on the core network.

[![v6mpls lab topology](diagram/v6mpls_lab.png)](diagram/v6mpls_lab.pdf?raw=true)

The IPv4 route PE1 learns from CE1 is announced to PE2 with a next-hop of the PE1 IPv6 loopback address.  The question we want to answer is what will PE2 on receipt of this route?  Given that the next-hop is an IPv6 address - _reachable via an MPLS path_ - will the route be installed?  And how will the data-plane encapsulation work on the wire for traffic to this destination?

## Protocol Notes

### MPLS Origins

It is easy to forget today that MPLS was developed before doing longest-prefix matches in hardware was possible, when telco's were still keen on technologies such as [ATM](https://en.wikipedia.org/wiki/Asynchronous_Transfer_Mode), with "cells" switched based on exact-matches and fixed-length IDs.  MPLS initially aimed to bridge this gap and provide a way for high-speed forwarding of IP packets using similar hardware.  [RFC 3031](https://www.rfc-editor.org/info/rfc3031/#section-2.1) alludes to this stating _"MPLS forwarding can be done by switches which are capable of doing label lookup and replacement, but are either not capable of analyzing the network layer headers, or are not capable of analyzing the network layer headers at adequate speed."_

By the time MPLS was being widely deployed hardware had begun to catch up, and high-speed IP routing ASICs were becoming available.  Nonetheless I believe this legacy is worth bearing in mind when examining how MPLS operates.  Many design choices seek to minimise the number of packet header inspections needed, or forwarding table lookups required.

### Ultimate Hops and Pops

This is not, in fact, a cheesy 80s hip hop compilation.

[Penultimate Hop Popping](https://www.rfc-editor.org/info/rfc3031/#section-3.16) defines a method by which the last 'P' router can **pop** the outer label from a packet before it is sent to the egress PE.  The RFC explains that this allows the egress PE *"to do a single lookup"*, because the packet doesn't arrive with an extra MPLS label for it to inspect.  The rationale for this optimisation is probably best considered in light of the previous section.

If the last P router *does not* remove the outer label we call it "ultimate hop popping" (UHP).  Meaning that label is instead popped at the _ultimate hop_, the egress PE.  There are two modes for UHP:

* *Explicit-null* means the last P router swaps the incoming label with a well-known generic one
** 0 is used if the LSP is built for an IPv4 prefix
** 2 is used if the LSP is built for an IPv6 prefix

So that leaves us with three possible ways the last P router can behave for an LSP.  Which it uses is based on how the PE device signals the LSP in the first place, with both LDP and SR supporting all three modes.

|No| Penultimate Action | P router label operation | PE LSP Signalling: LDP | PE LSP Signalling: SR IS-IS |
|--|---|---|---|---|
|1| **PHP / Implicit Null** | POP¹ | 3 (Implicit Null) | P=0 |
|2| **UHP / Explicit Null** | SWAP → 0 (IPv4) / 2 (IPv6) | 0 (IPv4 FEC) / 2 (IPv6 FEC) | P=1, E=1 |
|3| **UHP / non-null label** | SWAP → egress's label | Normal label (≥16) | P=1, E=0 |

NOTE: I've called this the "Penultimate Action" for the sake of this post, and given each a number, as we will need to refer to them a lot.
NOTE 2: The OSPFv3 flags for SR are similar, but "P" is called "NP".

## Labs

Currently there are four example lab topologies that can be run simply in containerlab.

### Images

The labs have containerlab 'kinds' that expect to find the following image names locally available in docker.  Users may need to "docker tag" the name of their equivalent images to match these, or update the YAML topology files to reference the names they have locally:

|Image Type|Image Name|
|----------|----------|
|Juniper vjunos-router|vrnetlab/juniper_vjunos-router:25.4R1-S2.3|
|Cisco xRD control plane|ios-xr/xrd-control-plane:latest|
|Arista cEOS|ceos:latest|
|Debian / FRR (CE)|debian:clab|

The linux image should have FRR installed.  The vanilla debian container image with _apt install frr_ should be sufficient.

### Lab topologies, configs and detailed test results.

The topology files, startup configs and results of testing can be found in the relevtant directories in this repo:

[Juniper vJunos Router - SR](sr-mpls-v6-junos)  
[Cisco xRD Control Plane](sr-mpls-v6-xrd)  
[Arista cEOS](sr-mpls-v6-arista)  
[Juniper vJunos Router - LDP](ldp-mpls-v6-junos)  

## Findings

More detailed information on how each lab performed is available in the above 

## Standards


Two more statements from RFC 3031 which will become relevant are:

_"In MPLS, the assignment of a particular packet to a particular FEC is done just once, as the packet enters the network."_

_"In the MPLS forwarding paradigm, once a packet is assigned to a FEC, no further header analysis is done by subsequent routers; all forwarding is driven by the labels."_

RFC3032 - contradictory about having to inspect further into packet?


So is it standards compliant?  No.

Linux?  

Standards don't mention it - but they probably couldn't imagine it.  Time of writing MPLS nobody expected todays scenario - that people would want to try to send IPv4 over an IPv6-only network.  The idea was we would migrate from IPv4 to IPv6 in a few years, and for those few years we would dual-stack everything.  It is only today, with no expiry date for IPv4 on the horizon, that people are trying to do wonky things like v4-over-v6.

## Solutions

### 4PE

 is the name used for the type of network described in this post.  The name clearly alludes to [6PE](https://www.rfc-editor.org/info/rfc4798/) used to transmit IPv6 over an IPv4-only MPLS core.  There is a draft standard, [draft-mishra-idr-v4-islands-v6-core--10](https://datatracker.ietf.org/doc/html/draft-mishra-idr-v4-islands-v6-core-4pe-10)), titled "Connecting IPv4 Islands over IPv6 Core using IPv4 Provider Edge Routers (4PE)" in an attempt to standardise the setup.

It presents various potential ways to solve the problem.  The options in section 8.1, 8.2 and 8.3 mirror 6PE, using [BGP labelled unicast](https://datatracker.ietf.org/doc/html/rfc8277) to send a label with each IPv4 BGP route announced between PEs.  In each case the ingress PE applies two labels to the IPv4 packet, the first from the BGP route, the second from the MPLS LSP.  While that neatly solves the issue I am here more interested in if the problem can be addressed without putting labels in BGP.

Section 8.4 of the 4PE draft describes a scenario that operates without BGP-LU.  PHP is not used on the penultimate P, so the packet arrives at the egress PE with the transport label intact.  The egress PE pops the transport label, then needs to inspect the remaining packet protocol version (IPv4 or IPv6), and then look up the destination IP in the appropriate routing table.  It cannot infer the address family or table for IP route lookup from the label.

Section 8.5 of the 4PE draft describes a similar setup, but with IPv6 explicit-null (2) signalled by the egress PE for the v6 loopback label.  This scenario is similar to what we observed in the Juniper lab with explicit null enabled.  The packet uses the single-label transport across the core, then at the last P router the label is swapped for '2' and it is forwarded to the PE.  This packet is technically invalid, label=2, S(last label)=1, payload=IPv4.  Only IPv6 should be contained within such a packet.  Juniper PEs as we seen fail to process this packet arriving.  Arista we observed does not assume label=2 means IPv6, despite that being in the standards, and instead inspects the packet as it would for the 4PE mode in section 8.4.

### Alternative options?

I believe there are other options beyond those envisioned in the 4PE draft, which may or may not be more practical.

1 - Implicit null signalled for IPv6 loopback by PE, no BGP-LU, with P inspecting IP version

The IPv6 prefix label is signalled with label=3 / implicit null requested and PHP enabled.  With this option the ingress PE applies a single label, of the LSP to the egress PE loopback, and uses it for both v4 and v6 packets.  No inner label is added.

The last P router pops this label as the LSP is built with implicit null.  What remains is an IPv6 or IPv4 packet, the P router cannot know which type based on the ingress label alone as the same label is used for both.  So the P router needs to:

* Use the ingress label to determine the outgoing interface and next-hop MAC address
** It only knows IGP routes so can't use the destination IP to find this
* Inspect the packet to see what IP version it is, and set the Ethertype on the frame based on this

The PE router in this case also needs to be able to deal with a frame arriving on an interface with no IPv4 addressing that has an IPv4 Ethertype value.

2 - Add explicit null labelling on ingress PE based on address fam, no BGP-LU

In this case the LSP can be set up for any mode of operation (no PHP, explicit null, implicit null).  In each of those cases it will operate as described in the 4PE draft sections 8.1.2, 8.2.2 and 8.3.2 respectively.

The difference is that each of those options in the draft use BGP-LU to announce the IPv4 routes to PEs with the label of '0'.  It should be possible, instead of using BGP-LU, to always impose the label of 0 based on the fact the packet is IPv4.

In this mode, for symmetry, I think it would make sense for the ingress PE to do the same and impose an inner label of '2' for all IPv6 packets.  






MAC known from ND.  Etype known from packet inspection.

Conceptually 



### VPNv4

### Standardise Arista's approach

The cleanest way 

- Ingress PE pushes two labels like in 4PE/VPNv4
- We don't exchange labels with MPLS though, the ingress PE always pushes 0 for an IPv4 packet or 2 for IPv6, with the S-bit set to 1.
- The transport label is pushed on top of this by the ingress PE



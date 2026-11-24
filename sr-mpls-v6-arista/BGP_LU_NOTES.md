# sr-mpls-v6-arista: BGP labelled-unicast for IPv4

Testing whether BGP labelled-unicast (RFC 8277, SAFI 4) between the PEs can carry IPv4 over the IPv6-only SR-MPLS core.  This is the approach in sections 8.1-8.3 of the [4PE draft](https://www.ietf.org/archive/id/draft-mishra-idr-v4-islands-v6-core-4pe-10.html).  The IPv4 routes stay in the global table; no VPNv4 or VRFs are used.  Image is cEOSLab 4.36.3M.

IPv4 already works on Arista without labelled-unicast (see [PHP_MODES.md](PHP_MODES.md)), so this was mainly to compare with the other vendors.

## Summary

**Not usable on cEOS 4.36.3M** in this topology, for two reasons:

1. **No IPv6 next-hop for IPv4 labelled-unicast.**  IPv4 unicast uses `neighbor IBGP next-hop address-family ipv6 originate` (RFC 8950), but that option doesn't exist under `address-family ipv4 labeled-unicast`.  Without an IPv4 source the PE can't send updates at all (`missing (cannot send updates with next hop self)`).  With an IPv4 source interface the next-hop is IPv4 and can't be resolved in the IPv6-only core.  An inbound route-map with `set ipv6 next-hop` didn't change the next-hop either.
2. **CE-learned routes don't get into labelled-unicast.**  A `network` statement under `address-family ipv4 labeled-unicast` picked up a connected route, but not CE2's loopback, which PE2 learned over eBGP IPv4 unicast.

The session negotiates the family fine, and labels are allocated for locally originated routes (`label local-termination implicit-null` by default, with `explicit-null` as an option).  The problem is the next-hop.

## Enabling labelled-unicast

On both PEs the IBGP peer group was moved from IPv4 unicast to IPv4 labelled-unicast:
```
router bgp 64496
   address-family ipv4
      no neighbor IBGP activate
   address-family ipv4 labeled-unicast
      neighbor IBGP activate
      neighbor IBGP next-hop-self
```
The session negotiates the family:
```
pe1#show bgp summary
BGP summary information for VRF default
Router identifier 203.0.113.101, local AS number 64496
Neighbor                AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
198.51.100.2         64497 Established   IPv4 Unicast            Negotiated              1          1          0
2001:db8:10::3       64496 Established   IPv4 MplsLabel          Negotiated              0          0          0
2001:db8:10::3       64496 Established   IPv6 Unicast            Negotiated              1          1          1
2001:db8:50::3       64497 Established   IPv6 Unicast            Negotiated              1          1          1
```
The defaults for the family (`show running-config all`, trimmed to the IBGP and global lines):
```
router bgp 64496
   address-family ipv4 labeled-unicast
      no bgp missing-policy direction in action
      no bgp missing-policy direction out action
      bgp additional-paths receive
      no bgp additional-paths send
      no bgp next-hop-unchanged
      no neighbor default next-hop-self
      no next-hop resolution disabled
      next-hop resolution ribs tunnel-rib colored system-colored-tunnel-rib tunnel-rib system-tunnel-rib system-connected
...
      neighbor IBGP activate
      neighbor IBGP additional-paths receive
      default neighbor IBGP graceful-restart
      no neighbor IBGP route-map in
      no neighbor IBGP route-map out
      no neighbor IBGP rcf in
      no neighbor IBGP rcf out
      no neighbor IBGP additional-paths send
      no neighbor IBGP next-hop-unchanged
      no neighbor IBGP next-hop-self
      no neighbor IBGP next-hop-self source-interface
      default neighbor IBGP aigp-session
      default neighbor IBGP multi-path
...
      no lfib entry installation skipped
      label local-termination implicit-null
      aigp-session ibgp
      aigp-session confederation
      no aigp-session ebgp
```

## Next-hop

The IPv6 next-hop option used for IPv4 unicast is rejected under labelled-unicast:
```
router bgp 64496
   address-family ipv4 labeled-unicast
      neighbor IBGP next-hop address-family ipv6 originate
```
```
> neighbor IBGP next-hop address-family ipv6 originate

% Invalid input at line 4
```
The BGP neighbor output shows EOS has no next-hop it can use for the labelled family:
```
pe2#show bgp neighbors 2001:db8:10::1
...
    Multiprotocol IPv4 with MPLS Labels: advertised and received and negotiated
...
Local TCP address is 2001:db8:10::3, local port is 179
Remote TCP address is 2001:db8:10::1, remote port is 33257
Local next hop for next hop self:
  IPv4 with MPLS Labels: missing (cannot send updates with next hop self)
  IPv6 Unicast: 2001:db8:10::3
```
`neighbor IBGP next-hop-self source-interface Loopback0` (Loopback0 only has an IPv6 address) still gives `missing`.  Using the CE-facing interface, which has an IPv4 address, provides a next-hop:
```
router bgp 64496
   address-family ipv4 labeled-unicast
      neighbor IBGP next-hop-self source-interface Ethernet1/1
```
```
Local next hop for next hop self:
  IPv4 with MPLS Labels: 198.51.100.5
```

## Getting routes into labelled-unicast

`network 203.0.113.2/32` (CE2's loopback, learned over eBGP IPv4 unicast) under the labelled-unicast family didn't create any route:
```
pe2#show bgp ipv4 labeled-unicast detail
BGP routing table information for VRF default
Router identifier 203.0.113.103, local AS number 64496
```
`network 198.51.100.4/30`, the connected CE2 link, did, with local label 3:
```
pe2#show bgp ipv4 labeled-unicast detail
BGP routing table information for VRF default
Router identifier 203.0.113.103, local AS number 64496
BGP routing table entry for 198.51.100.4/30
 Paths: 1 available
  Local
    - labels [ ] from - (203.0.113.103)
      Origin IGP, metric -, localpref -, IGP metric -, weight 0, tag 0
      Received 00:00:00 ago, valid, local, best, redistributed (Connected)
      Local MPLS label: 3
      Rx SAFI: MplsLabel
      Tunnel RIB eligible
```
PE1 receives it with PE2's IPv4 address as next-hop, which can't be resolved, so it's invalid:
```
pe1#show bgp ipv4 labeled-unicast detail
BGP routing table information for VRF default
Router identifier 203.0.113.101, local AS number 64496
BGP routing table entry for 198.51.100.4/30
 Paths: 1 available
  Local
    198.51.100.5 labels [ 3 ] from 2001:db8:10::3 (203.0.113.103)
      Origin IGP, metric 0, localpref 100, IGP metric -, weight 0, tag 0
      Received 00:00:05 ago, invalid, internal
      Rx SAFI: MplsLabel
      Tunnel RIB eligible
```

## Next-hop rewrite attempt

On Junos, an import policy setting the next-hop to the sending PE's IPv6 loopback worked around this.  The EOS equivalent was accepted, but had no effect, even after a soft inbound clear:
```
route-map LU-NH-V6 permit 10
   set ipv6 next-hop 2001:db8:10::3
!
router bgp 64496
   address-family ipv4 labeled-unicast
      neighbor IBGP route-map LU-NH-V6 in
```
```
pe1#clear bgp 2001:db8:10::3 soft in
```
```
pe1#show bgp ipv4 labeled-unicast detail
BGP routing table information for VRF default
Router identifier 203.0.113.101, local AS number 64496
BGP routing table entry for 198.51.100.4/30
 Paths: 1 available
  Local
    198.51.100.5 labels [ 3 ] from 2001:db8:10::3 (203.0.113.103)
      Origin IGP, metric 0, localpref 100, IGP metric -, weight 0, tag 0
      Received 00:00:21 ago, invalid, internal
      Rx SAFI: MplsLabel
      Tunnel RIB eligible
```
Since no route ever became valid, label allocation (per prefix vs per next-hop vs explicit null) couldn't be tested end to end.

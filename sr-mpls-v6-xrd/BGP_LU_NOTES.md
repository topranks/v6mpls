# sr-mpls-v6-xrd: BGP labelled-unicast for IPv4

Testing whether BGP labelled-unicast (RFC 8277, SAFI 4) between the PEs can carry IPv4 over the IPv6-only SR-MPLS core.  This is the approach in sections 8.1-8.3 of the [4PE draft](https://www.ietf.org/archive/id/draft-mishra-idr-v4-islands-v6-core-4pe-10.html).  The IPv4 routes stay in the global table; no VPNv4 or VRFs are used.

## Summary

**This can't be configured on XRd 24.2.11.**  IOS XR accepts `address-family ipv4 unicast` on an IPv6 neighbor (RFC 8950, which is how the lab works today).  It does not accept `address-family ipv4 labeled-unicast` on an IPv6 neighbor at all.  There is no IPv4 addressing between the PEs to build an IPv4 iBGP session instead, so the labelled-unicast approach is not available on this platform.

RFC 8950 defines IPv4 NLRI with an IPv6 next-hop for labelled-unicast (AFI 1 / SAFI 4) as well as unicast, so this is an implementation limitation, not a standards one.

```
RP/0/RP0/CPU0:pe1#show version
Cisco IOS XR Software, Version 24.2.11 LNT
Copyright (c) 2013-2024 by Cisco Systems, Inc.

Build Information:
 Built By     : cisco
 Built On     : Tue Jul 02 14:32:37 UTC 2024
 Build Host   : iox-ucs-073
 Workspace    : /auto/srcarchive11/prod/24.2.11/xrd-control-plane/ws/
 Version      : 24.2.11
 Label        : 24.2.11

cisco XRd Control Plane
cisco XRd-CP-C-01 processor with 32GB of memory
pe1 uptime is 2 minutes
XRd Control Plane Container
```

## What was tried

Replacing `ipv4 unicast` with `ipv4 labeled-unicast` in the IBGP neighbor-group, plus `allocate-label all` (needed on XR for BGP to allocate labels for eBGP-learned routes):
```
router bgp 64496
 address-family ipv4 unicast
  allocate-label all
 exit
 neighbor-group IBGP
  no address-family ipv4 unicast
  address-family ipv4 labeled-unicast
   next-hop-self
```
The labelled-unicast part is rejected because the neighbor using the group is IPv6:
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
Configuring it directly on the neighbor gives a syntax error.  The address family doesn't exist under an IPv6 neighbor:
```
router bgp 64496
 neighbor 2001:db8:10::3
  address-family ipv4 labeled-unicast
```
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
(The same result on PE2 for neighbor 2001:db8:10::1.)

With `ipv4 unicast` on the same IPv6 neighbor the config is accepted (this is the lab's startup config), so XR clearly supports RFC 8950 for SAFI 1 but not SAFI 4.

## Related

In [PHP_MODES.md](PHP_MODES.md), IPv4 failed on XRd in every PHP mode, because PE1 never transmitted IPv4 packets onto the IPv6 LSP.  The Junos lab later showed that adding `family inet` (no address) to the core interfaces made implicit null and non-null UHP work there (see [../sr-mpls-v6-junos/BGP_LU_NOTES.md](../sr-mpls-v6-junos/BGP_LU_NOTES.md#addendum-family-inet-on-core-interfaces)).  The XR equivalent would be enabling IPv4 on the core interfaces (e.g. `ipv4 unnumbered Loopback0`, which would need an IPv4 loopback address).  That wasn't tested here, as it adds IPv4 addressing to the core.

# Task 20 introduction to BGP overlay

By now we should have an overview for the basics of BGP - underlay. Advertising client subnets from `leaf1` to `leaf2` and viceversa. 

In order to get one step closer to the EVPN/VXLAN fabrics, we will first configure BGP overlay on top of the BGP underlay (iBGP on top of eBGP). 
In this exersise we will use overlay to carry IPv4 routes on top of existing underlay.

The goal for this exercise is to move the existing `client1` and `client2` from underlay BGP to overlay BGP.

## Overlay bgp
In SR Linux-based EVPN/VXLAN fabrics, “overlay BGP” refers to the BGP sessions that carry EVPN routes (and sometimes VPNv4/v6) on top of an IP underlay.

Key points from the [`references`](https://learn.srlinux.dev/tutorials/l3evpn/rt5-only/underlay/):

Underlay vs overlay roles

- Underlay BGP (usually eBGP) advertises loopbacks / system0 addresses and provides pure IP reachability. 
- Overlay BGP (iBGP or eBGP) carries EVPN (and optionally L3VPN) routes between VTEPs to build L2/L3 services. 


Let's start moving the `client1` and `client2` from being advertised into eBGP to be advertised into iBGP. For this we will need an iBGP session between `leaf` and `leaf2` system addresses.

### Task 20.1: Create system0 subinterface 0 and allocate a ipv4 addresses, also make sure it is added to the default network routing instance
- 10.0.0.1/32 will be used on `leaf1`
- 10.0.0.2/32 will be used on `leaf2`

<details>
<summary>Task 20.1 solution</summary>

`Leaf1`

```
insert / interface system0 subinterface 0 ipv4 admin-state enable
insert / interface system0 subinterface 0 ipv4 address 10.0.0.1/32

insert / network-instance default interface system0.0
```

`Leaf2`

```
insert / interface system0 subinterface 0 ipv4 admin-state enable
insert / interface system0 subinterface 0 ipv4 address 10.0.0.2/32

insert / network-instance default interface system0.0
```
</details>

### Task 20.2: Update the existing policies `export-underlay` and `import-underlay`, each with a new statment named `accept-loopbacks`.

<details>
<summary>Task 20.2 solution</summary>

`Leaf1`

```
insert / routing-policy prefix-set loopbacks prefix 10.0.0.0/24 mask-length-range 32..32

insert / routing-policy policy export-underlay statement accept-loopbacks after accept-clients
insert / routing-policy policy export-underlay statement accept-loopbacks match protocol local
insert / routing-policy policy export-underlay statement accept-loopbacks match prefix prefix-set loopbacks
insert / routing-policy policy export-underlay statement accept-loopbacks action policy-result accept

insert / routing-policy policy import-underlay statement accept-loopbacks after accept-clients
insert / routing-policy policy import-underlay statement accept-loopbacks match prefix prefix-set loopbacks
insert / routing-policy policy import-underlay statement accept-loopbacks action policy-result accept
```

`Leaf2`

```
insert / routing-policy prefix-set loopbacks prefix 10.0.0.0/24 mask-length-range 32..32

insert / routing-policy policy export-underlay statement accept-loopbacks after accept-clients
insert / routing-policy policy export-underlay statement accept-loopbacks match protocol local
insert / routing-policy policy export-underlay statement accept-loopbacks match prefix prefix-set loopbacks
insert / routing-policy policy export-underlay statement accept-loopbacks action policy-result accept

insert / routing-policy policy import-underlay statement accept-loopbacks after accept-clients
insert / routing-policy policy import-underlay statement accept-loopbacks match prefix prefix-set loopbacks
insert / routing-policy policy import-underlay statement accept-loopbacks action policy-result accept
```

Loopbacks should be visible in the routing table and the `leafs` should be able to ping between loopback interfaces
```
 A:admin@leaf1# ping network-instance default -I 10.0.0.1 10.0.0.2
 Using network instance default
 PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 : 56(84) bytes of data.
 64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=5.02 ms
```
</details>

Loopbacks should be visible in the routing table and the `leafs` should be able to ping between loopback (system) interfaces.




### Task 20.3: Delete existing policy export-underlay statement accept-clients and policy import-underlay statement accept-clients. 
> [!NOTE]
> By removing these policies we remove the advertisment of the `client1` and `client2` into the underlay BGP (eBGP). The ping between clients should not be possible anymore at this point, but not a problem, we want to move these subnet to iBGP.


<details>
<summary>Task 20.3 solution</summary>

`Leaf1`

```
delete / routing-policy policy export-underlay statement accept-clients
delete / routing-policy policy import-underlay statement accept-clients
```

`Leaf2`

```
delete / routing-policy policy export-underlay statement accept-clients
delete / routing-policy policy import-underlay statement accept-clients
```

</details>

### Task 20.4: Create iBGP sessions between `leaf1` and `leaf2` system ip addresses using ASN 65000, and advertise the `clients` prefixes from one leaf to another. The new iBGP sessions should be part of group `ibgp-overlay`. 

> [!TIP] 
> iBGP session are configured in the same way as eBGP, as long as `leaf1` and `leaf2` will use the same ASN, they will establish an iBGP session.
> Create new `export-overlay` and `import-overlay` policies with statment `accept-clients` matching on the `client` prefixes.
> The local ASN should be rewritten to ASN65000 into the peer group.

Check the BGP neighbors, the new iBGP session should be in established state.
Check the routing table on each leaf, we should see the client routes present into the routing table, with next-hop the system IP of the other leaf. 

<details>
<summary>Task 20.4 solution</summary>

`Leaf1`

```
insert / network-instance default protocols bgp group ibgp-overlay peer-as 65000
insert / network-instance default protocols bgp group ibgp-overlay export-policy [ export-overlay ] first
insert / network-instance default protocols bgp group ibgp-overlay import-policy [ import-overlay ] first
insert / network-instance default protocols bgp group ibgp-overlay local-as as-number 65000
insert / network-instance default protocols bgp group ibgp-overlay transport local-address 10.0.0.1
insert / network-instance default protocols bgp neighbor 10.0.0.2 peer-group ibgp-overlay

insert / routing-policy policy export-overlay statement accept-clients first
insert / routing-policy policy export-overlay statement accept-clients match protocol local
insert / routing-policy policy export-overlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy export-overlay statement accept-clients action policy-result accept

insert / routing-policy policy import-overlay statement accept-clients first
insert / routing-policy policy import-overlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy import-overlay statement accept-clients action policy-result accept
```

`Leaf2`

```
insert / network-instance default protocols bgp group ibgp-overlay peer-as 65000
insert / network-instance default protocols bgp group ibgp-overlay export-policy [ export-overlay ] first
insert / network-instance default protocols bgp group ibgp-overlay import-policy [ import-overlay ] first
insert / network-instance default protocols bgp group ibgp-overlay local-as as-number 65000
insert / network-instance default protocols bgp group ibgp-overlay transport local-address 10.0.0.2
insert / network-instance default protocols bgp neighbor 10.0.0.1 peer-group ibgp-overlay

insert / routing-policy policy export-overlay statement accept-clients first
insert / routing-policy policy export-overlay statement accept-clients match protocol local
insert / routing-policy policy export-overlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy export-overlay statement accept-clients action policy-result accept

insert / routing-policy policy import-overlay statement accept-clients first
insert / routing-policy policy import-overlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy import-overlay statement accept-clients action policy-result accept
```

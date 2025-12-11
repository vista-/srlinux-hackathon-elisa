# Section 40: Creating L2 EVPN services

## Task 40.1: Adding a third client
Now that we are done building an IP fabric, let's start creating services on top of our robust network.

First, let's make things a bit more interesting in our topology by adding yet another client.

>[!IMPORTANT]
> Don't forget to save your work on the spine switches to new startup-configuration files, and the additional work you have done on the leaf switches!

`client3` should be connected to `ethernet-1/2` of `leaf2` in our topology, and it should be yet another `network-multitool` type client. No need to configure an IP on this one just yet!

## Task 40.2: VLANs on SR Linux

After we deploy our newest topology, we have to get acquainted with a new concept: VLANs!

VLANs on SR Linux are not locally significant to the switch, they are locally significant to the subinterface, and merely define a (list of) VLAN tags to pop and push on ingress and egress, respectively.

On the same interface, a subinterface can be untagged ("access VLAN", or "native VLAN" in case other tagged traffic is also present), single-tagged (where the tag can be a single ID, or a range of IDs, or even "any"), or even double-tagged on select platforms. We have documented the behaviour of VLANs on SR Linux in this [dedicated comprehensive article](https://learn.srlinux.dev/blog/2024/vlans-on-sr-linux).

So far, `ethernet-1/1` only had a single subinterface (ID '0'), which was a routed subinterface.

Let's replace this, by changing our client connectivity setup! From now on, all traffic between the client and the SR Linux leaf switches via `ethernet-1/1` (and `ethernet-1/2` on `leaf2`) will be tagged with a VLAN. We will start with the first VLAN, VLAN 100, which will be our plain L2VPN service.

**Configure the VLAN 100 bridged subinterface on the SR Linux leafs, [configure VLAN tagging on the Linux client interface side](https://wiki.archlinux.org/title/VLAN), and number the client interfaces from the 192.168.100.0/24 address range** (note: the leaf switches require no addressing).  
You can also delete the routed subinterface that we used so far.

<details>
<summary>Task 40.2 solution</summary>

Leaf config:
```
delete / interface ethernet-1/1 subinterface 0
insert / interface ethernet-1/1 vlan-tagging true
insert / interface ethernet-1/1 subinterface 100 type bridged
insert / interface ethernet-1/1 subinterface 100 vlan encap single-tagged vlan-id 100
# Repeat for ethernet-1/2 for leaf2
```

Client config:
```
ip link add link eth1 name eth1.100 type vlan id 100
ip link set dev eth1.100 up
ip addr add 192.168.0.1/24 dev eth1.100
# Repeat for other clients
```
</details>

Can you ping between clients? Even between `client2` and `client3`, connected to the same leaf switch?  

## Task 40.3: MAC-VRFs

If you came to the conclusion "No" in the last task, then you are absolutely right!

VLANs on SR Linux behave similarly to SR OS, and they don't behave like on many other network OSes: two subinterfaces configured with the same VLAN do not implicitly belong to the broadcast domain.  
Instead, you will need to assign these two subinterfaces to the same network-instance in order to have them in the same broadcast domain.

You could assign them to the `default` network-instance, but where's the fun in that? Let's leave the `default` network-instance to do its job as our control plane, and instead, define an L2 service using a separate network-instance, like it should be!

This is where MAC-VRFs come in! As a recap, MAC-VRFs are one of the types that network-instances can be, and these are bridged network-instances. Analogues to the MAC-VRF term is bridge domain, which might be more familiar from other network OSes.

**Create a new MAC-VRF network-instance called "l2" on both leaf switches, and assign the client subinterfaces to it!** Try pinging between clients again!

<details>
<summary>Task 40.3 solution</summary>

Configuring a MAC-VRF on `leaf1`:
```
insert / network-instance l2 type mac-vrf
insert / network-instance l2 interface ethernet-1/1.100
# repeat on leaf2, incl. ethernet-1/2.100
```

Pings between `client2` and `client3` work, but not between `client1` and other clients.
</details>

## Task 40.4: Creating your first VXLAN Tunnel Endpoint

We have the first real task for our overlay network - let's use it to carry EVPN routes to interconnect services! But first, we need to create said EVPN routes. To do this, we first need to create some place for the overlay transport, in our data center use case, VXLAN, to terminate in and originate from. This is where VXLAN tunnel endpoints (VTEPs) come into play.

**For each service (VNI) we want to create in SR Linux, we need to create a VXLAN tunnel subinterface, and bind a VNI ID to it.** The VNI in our case should be the same as the VLAN ID, 100. You can use the [SR Linux VXLAN guide](https://documentation.nokia.com/srlinux/25-10/books/vpn-services/vxlan-v4.html) to create this tunnel endpoint and bind it to the MAC-VRF we just created.

<details>
<summary>Task 40.4 solution part 1</summary>

Configuring a VXLAN Tunnel Endpoint (VTEP) and assigning it to the "l2" MAC-VRF:
```
## First, create VXLAN tunnel, VNI = 100
insert / tunnel-interface vxlan0 vxlan-interface 100 type bridged
insert / tunnel-interface vxlan0 vxlan-interface 100 ingress vni 100
## Associate VXLAN tunnel with MAC-VRF
insert / network-instance l2 vxlan-interface vxlan0.100
insert / network-instance l2 protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan0.100
```

</details>

SR Linux also lets you adjust the behaviour for both BGP EVPN routes are formed and how the VPNv4 BGP family is used to import and export routes via route targets and distinguishers.  
**Let's set the EVI to the same value as the VNI, leave the RD to be auto-derived, and the route-target we use should be `target:65000:EVI`!**

<details>
<summary>Task 40.4 solution part 2</summary>

Configuring BGP-EVPN and BGP-VPN parameters for the "l2" MAC-VRF:
```
## Set EVPN Instance ID to be the same as the VNI
insert / network-instance l2 protocols bgp-evpn bgp-instance 1 evi 100
## Manually set RT, as it is auto-derived from the ASN, and we have diff ASN per leaf
insert / network-instance l2 protocols bgp-vpn bgp-instance 1 route-target export-rt target:65000:100
insert / network-instance l2 protocols bgp-vpn bgp-instance 1 route-target import-rt target:65000:100
```

</details>

You should be able to see the VTEP interface in `show tunnel-interface vxlan0 vxlan-interface 100` and `show network-instance l2 vxlan-interface`.

After this is done... we can't ping yet! This is because we haven't enabled the EVPN address family in BGP yet. We will solve this in the next task.

## Task 40.5: Enabling EVPN in BGP

As we discussed earlier, address families must be enabled manually in SR Linux's BGP configuration in order for those routes to be imported into the BGP RIB.

Our task here will be two-fold: **turn our _overlay_ BGP sessions into a true _overlay_ by disabling IPv4 and enabling EVPN address families, and also by replace the import/export policies to reflect this!**. Here's a good [guide](https://learn.srlinux.dev/tutorials/l2evpn/evpn/#evpn-configuration) to get you started.

Make sure to do the changes across all nodes!

<details>
<summary>Task 40.5 solution</summary>

Configuring BGP-EVPN and BGP-VPN parameters for the "l2" MAC-VRF:
```
## Enable EVPN and disable IPv4 address family in the overlay group
insert / network-instance default protocols bgp group ibgp-overlay afi-safi evpn admin-state enable
insert / network-instance default protocols bgp group ibgp-overlay afi-safi ipv4-unicast admin-state disable

## Set EVPN routes to be distributed in overlay export and import policies
delete / routing-policy policy export-overlay
delete / routing-policy policy import-overlay  
insert / routing-policy policy export-overlay statement accept-evpn first
insert / routing-policy policy export-overlay statement accept-evpn match family [ evpn ]
insert / routing-policy policy export-overlay statement accept-evpn action policy-result accept
insert / routing-policy policy import-overlay statement accept-evpn first
insert / routing-policy policy import-overlay statement accept-evpn match family [ evpn ]
insert / routing-policy policy import-overlay statement accept-evpn action policy-result accept
```

</details>

You should be able to ping between `client1` and `client2` now!

You can also use the `show network-instance default protocols bgp routes evpn route-type summary` to look at the EVPN routes received in the overlay peering session, and the `show network-instance l2 bridge-table mac-table all` command to look at the bridge table of the "l2" service. The remote MAC will be shown as having a VXLAN tunnel interface as a destination, with the VTEP and the VNI marked.  
To show only the remote MACs learned in VNI 100, use the `show tunnel-interface vxlan0 vxlan-interface 100 bridge-table mac-table` command.
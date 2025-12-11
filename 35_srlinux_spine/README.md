# Configuring the spines in our data center

## Task 35.1: Defaults in Containerlab

Our Containerlab topology is getting a bit busy! Thankfully, we can use defaults to simply our topology description further, before we deploy the topology once again.

These can be set directly under the `topology` section of the Containerlab topology file, and thus, are [documented in the topology definition](https://containerlab.dev/manual/topo-def-file/). For example, if all (or most) of the nodes in the topology are Nokia SR Linux Kinds, it might make sense to make that Kind the default value.

Additionally, there is also the ability to set per-Kind defaults under the `kinds` section of the Containerlab topology definition. For example, to set the default image for each Kind. This is documented also in the topology definition.

Your next task is to simplify the topology file a bit by making the default Kind in the topology to Nokia SR Linux, set the default image for both the Nokia SR Linux and the Linux Kind, and setting each SR Linux node to be a IXR-D3L hardware variant by default.

<details>
<summary>Task 35.1 solution</summary>

```yaml
name: hackathon

topology:
  defaults:
    kind: nokia_srlinux
  kinds:
    nokia_srlinux:
      type: ixr-d3l
      image: ghcr.io/nokia/srlinux:25.10.1
    linux:
      image: ghcr.io/srl-labs/network-multitool
  nodes:
    leaf1:
      startup-config: config/leaf1.cli
    leaf2:
      startup-config: config/leaf2.cli
    spine1:
      startup-config: config/spine1.cli
      type: ixr-h4-32d
    spine2:
      startup-config: config/spine2.cli
      type: ixr-h4-32d
    client1:
      kind: linux
  # ...
```

</details>

## Task 35.2: Configuring the new spine switches
Great, we added the two new spines to the topology, simplified it, and now deployed it!

But we still don't have these switches participating in the actual network we have built and configured ourselves. Let's fix that, we already know how!

**Our TODO-list for this task is the following:**

- Configure interfaces between spines and leaves. Let's use 10.0.10.0/24 to allocate /31s P2P nets out of!
- Configure system interfaces (use 10.0.0.11/32 and 10.0.0.12/32)
- Assign them to the default network-instance
- Bring up the underlay BGP group and peering sessions. Following the [Nokia Validated Design](https://www.nokia.com/ip-networks/validated-designs/) recommendations, use the same ASN (65100) on both spine switches

> [!NOTE]
>
> Why is this the recommended way of numbering ASNs in a leaf-spine fabric? We suggest digging a bit into the topic to understand this design decision!
> <details>
> <summary> Solution </summary>
> Using the same ASN for the spines gives you [_valley-free routing_](https://blog.ipspace.net/2018/09/valley-free-routing) for free, and prevents BGP path hunting, which can prove to be an issue in larger datacenter fabrics.
> </details>

- Configure the overlay BGP group and sessions to redistribute BGP routes (export and import all BGP routes). The spines should act as iBGP route-reflectors (again, following the NVD), and should set next-hop self -- these can be set as knobs on the peering sessions and groups.
- Don't forget to also configure the import and export policies!
- Configure the other side of the interfaces' on the leaf switches, and bring up the BGP peerings there as well.

<details>
<summary>Task 35.4 solution</summary>

One of the spines' example configuration can be found here:
```
insert / interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
insert / interface ethernet-1/1 subinterface 0 ipv4 address 10.0.10.0/31
insert / interface ethernet-1/2 subinterface 0 ipv4 admin-state enable
insert / interface ethernet-1/2 subinterface 0 ipv4 address 10.0.10.4/31

insert / interface system0 subinterface 0 ipv4 admin-state enable
insert / interface system0 subinterface 0 ipv4 address 10.0.0.11/32

insert / network-instance default interface ethernet-1/1.0
insert / network-instance default interface ethernet-1/2.0
insert / network-instance default interface system0.0

# BGP underlay
## Same AS number for the two spines!
insert / network-instance default protocols bgp autonomous-system 65100
insert / network-instance default protocols bgp router-id 10.0.0.11
insert / network-instance default protocols bgp afi-safi ipv4-unicast admin-state enable
insert / network-instance default protocols bgp group ebgp-underlay export-policy [ export-underlay ] first
insert / network-instance default protocols bgp group ebgp-underlay import-policy [ import-underlay ] first
insert / network-instance default protocols bgp neighbor 10.0.10.1 peer-group ebgp-underlay
insert / network-instance default protocols bgp neighbor 10.0.10.1 peer-as 65001
insert / network-instance default protocols bgp neighbor 10.0.10.5 peer-group ebgp-underlay
insert / network-instance default protocols bgp neighbor 10.0.10.5 peer-as 65002

## Policy for underlay
insert / routing-policy prefix-set loopbacks prefix 10.0.0.0/24 mask-length-range 32..32

insert / routing-policy policy export-underlay statement accept-loopbacks first
insert / routing-policy policy export-underlay statement accept-loopbacks match protocol local
insert / routing-policy policy export-underlay statement accept-loopbacks match prefix prefix-set loopbacks
insert / routing-policy policy export-underlay statement accept-loopbacks action policy-result accept

## Additional statement to re-advertise received loopbacks
insert / routing-policy policy export-underlay statement accept-bgp after accept-loopbacks
insert / routing-policy policy export-underlay statement accept-bgp match protocol bgp
insert / routing-policy policy export-underlay statement accept-bgp action policy-result accept

insert / routing-policy policy import-underlay statement accept-loopbacks first
insert / routing-policy policy import-underlay statement accept-loopbacks match prefix prefix-set loopbacks
insert / routing-policy policy import-underlay statement accept-loopbacks action policy-result accept

# BGP overlay
insert / network-instance default protocols bgp group ibgp-overlay peer-as 65000
insert / network-instance default protocols bgp group ibgp-overlay export-policy [ export-overlay ] first
insert / network-instance default protocols bgp group ibgp-overlay import-policy [ import-overlay ] first
insert / network-instance default protocols bgp group ibgp-overlay local-as as-number 65000
## Route reflector on
insert / network-instance default protocols bgp group ibgp-overlay route-reflector client true 
insert / network-instance default protocols bgp group ibgp-overlay route-reflector cluster-id 10.0.0.0
## Next-hop self
insert / network-instance default protocols bgp group ibgp-overlay next-hop-self true
insert / network-instance default protocols bgp group ibgp-overlay transport local-address 10.0.0.11
insert / network-instance default protocols bgp neighbor 10.0.0.1 peer-group ibgp-overlay
insert / network-instance default protocols bgp neighbor 10.0.0.2 peer-group ibgp-overlay

# Reflect all w/ BGP policy
insert / routing-policy policy export-overlay statement accept-bgp first
insert / routing-policy policy export-overlay statement accept-bgp match protocol bgp
insert / routing-policy policy export-overlay statement accept-bgp action policy-result accept

insert / routing-policy policy import-overlay statement accept-bgp first
insert / routing-policy policy import-overlay statement accept-bgp match protocol bgp
insert / routing-policy policy import-overlay statement accept-bgp action policy-result accept
```

</details>

Verify that the routes between the leafs and spines are exchanged correctly (loopbacks via the underlay sessions, client subnets via the overlay sessions, next-hop rewritten on client subnets)!

## Task 35.3: Making a true leaf-spine topology
So far so good! However, we still have that dangling direct link between the leafs, which is still the preferred route for client traffic. **Let's disable that interface and ensure client connectivity is still working between `client1` and `client2`!**

<details>
<summary>Task 35.3 solution</summary>

On both leaf switches (using the appropriate neighbor IPs):
```
# Disable interface
insert / interface ethernet-1/30 admin-state disable
# Delete underlay eBGP session
delete / network-instance default protocols bgp neighbor 192.168.0.2
# Delete direct overlay iBGP session
delete / network-instance default protocols bgp neighbor 10.0.0.2
```
</details>


Now we are done converting our topology to a real leaf-spine topology.
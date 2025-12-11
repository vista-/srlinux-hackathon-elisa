# Section 45: Creating L3 EVPN services

We will be setting up two different kinds of L3 service: one using the same Route Type 2 EVPN routes we have used in building the L2 service, and one using only Route Type 5 EVPN routes, which only contain IP prefix information.

In the first case, the clients will be connected via routed interfaces, and will be directly connected to an IP-VRF.

In the second case, the clients will be in MAC-VRFs, and connected via bridged interfaces, and our switch will be acting as a router (gateway) to the clients inside the MAC-VRFs via the IRB (integrated routing-bridging) interface.

## Task 45.1: Routed interface L3 EVPN

Let's create the client interfaces, both on the SR Linux, and Linux side:

- Use VLANs 200 for all connections
- Client subnets should be `192.168.201.0/24`, `192.168.202.0/24` and `192.168.203.0/24` for `client1`, `client2` and `client3` respectively. SR Linux side should be .1, client side should be .2
- Clients should have a static route for `192.168.200.0/22` routed via the routed interface's IP

The created routed interfaces should be terminated in an IP-VRF called "routed", and the VNI used should be 200.

A useful resource for learning SR Linux is not only the official Nokia SR Linux documentation, but also the "Learn SR Linux" companion website, which contains quick start tutorials and short guides on getting certain use-cases up and running. As luck has it, there is a nice guide on [getting L3VPNs up on SR Linux](https://learn.srlinux.dev/tutorials/l3evpn/rt5-only/l3evpn/)!

Here are some of the key differences between an L2 and L3 EVPN service's setup, to help you on your configuration journey:
- Not only the network-instance type is going to be different! The VXLAN tunnel is also going to be routed instead of bridged
- You don't have to import routes manually into the EVPN instance to be advertised, much like learned neighbor MACs

<details>
<summary>Task 45.1 solution</summary>

On SR Linux:

```
insert / interface ethernet-1/1 subinterface 200 type routed
insert / interface ethernet-1/1 subinterface 200 vlan encap single-tagged vlan-id 200
insert / interface ethernet-1/1 subinterface 200 ipv4 admin-state enable
insert / interface ethernet-1/1 subinterface 200 ipv4 address 192.168.201.1/24
# Also ethernet-1/2 on leaf2

insert / network-instance routed type ip-vrf
insert / network-instance routed interface ethernet-1/1.200

insert / tunnel-interface vxlan0 vxlan-interface 200 type routed
insert / tunnel-interface vxlan0 vxlan-interface 200 ingress vni 200

insert / network-instance routed vxlan-interface vxlan0.200
insert / network-instance routed protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan0.200

insert / network-instance routed protocols bgp-evpn bgp-instance 1 evi 200
# RT has to be different!
insert / network-instance routed protocols bgp-vpn bgp-instance 1 route-target export-rt target:65000:200
insert / network-instance routed protocols bgp-vpn bgp-instance 1 route-target import-rt target:65000:200

```

On the clients:

```bash
ip link add link eth1 name eth1.200 type vlan id 200
ip link set dev eth1.200 up
ip addr add 192.168.201.2/24 dev eth1.200
ip route add 192.168.200.0/22 via 192.168.201.1

# ... repeat on other clients
```

</details>

You can verify that the routes are installed using the `show network-instance routed route-table` command. To check out the EVPN routes, use the `show network-instance default protocols bgp routes evpn route-type summary` command -- they will all be Route Type 5, as only the IP prefix information is advertised in this EVPN L3VPN.

We can also verify that the L3VPN works by either pinging from a client or from the switch directly.

## Task 45.2: IRB-based L3 EVPN

For the second example L3 EVPN, we are going to use the _asymmetric_ IRB model. This means that the ingress VTEP performs the routing, while the egress VTEP only performs bridging. This also means both the source and destination MAC-VRF must also exist on the ingress VTEP, even if the destination MAC-VRF is not present otherwise on the ingress VTEP switch.

To start off on these tasks, let's create the client interfaces, both on the SR Linux, and the Linux side!

We will be using VLAN 210 and 211. Let's set up the following:
- `client1` and `client2` will be in VLAN 210, `client3` will be in VLAN 211.
- `client1` and `client2` will be in the "l2-irb1" MAC-VRF, while `client3` will be in the "l2-irb2" MAC-VRF. The IP-VRF should be called "l3-irb"
- Clients should be addressed out of `192.168.210.0/24` in VLAN 210, `192.168.211.0/24` in VLAN 211. An IRB subinterface should be created with the address .254 for both VLANs.
- Clients should have a static route for `192.168.210.0/23` pointed to the gateway .254 in the same subnet.

We will use IRB interfaces to serve as our gateways, and these also need to exist on both leaf switches, otherwise the ingress switch will not be able to make the routing decision.

Notice that only a single gateway IP address is specified, .254, yet we need to apply this on both switches.  
How is this possible?

The IRBs must be configured as [anycast gateways](https://documentation.nokia.com/srlinux/25-10/books/vpn-services/evpn-vxlan-tunnels-layer-3.html#anycast-gateways).

To tie the two MAC-VRFs together, we use an IP-VRF - how do we connect a MAC-VRF to an IP-VRF? We put the IRB interface both into the MAC-VRF and IP-VRF. For an in-depth guide on how to do this, take a look at the [asymmetric IRB configuration guide](https://documentation.nokia.com/srlinux/25-10/books/vpn-services/evpn-vxlan-tunnels-layer-3.html#asymmetric-irb) on the SR Linux documentation page.

<details>
<summary>Task 45.2 solution</summary>

SR Linux configuration:
```
insert / interface ethernet-1/1 subinterface 210 type bridged
insert / interface ethernet-1/1 subinterface 210 vlan encap single-tagged vlan-id 210
# Also ethernet-1/2 on leaf2

# VLAN 210 / l2-irb1
insert / tunnel-interface vxlan0 vxlan-interface 210 type bridged
insert / tunnel-interface vxlan0 vxlan-interface 210 ingress vni 210

insert / interface irb0 subinterface 210 ipv4 admin-state enable
insert / interface irb0 subinterface 210 ipv4 address 192.168.210.254/24 anycast-gw true
insert / interface irb0 subinterface 210 anycast-gw

insert / network-instance l2-irb1 type mac-vrf
insert / network-instance l2-irb1 interface ethernet-1/1.210
insert / network-instance l2-irb1 vxlan-interface vxlan0.210
insert / network-instance l2-irb1 protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan0.210
insert / network-instance l2-irb1 protocols bgp-evpn bgp-instance 1 evi 210
insert / network-instance l2-irb1 protocols bgp-vpn bgp-instance 1 route-target export-rt target:65000:210
insert / network-instance l2-irb1 protocols bgp-vpn bgp-instance 1 route-target import-rt target:65000:210

# VLAN 211 / l2-irb2
insert / tunnel-interface vxlan0 vxlan-interface 211 type bridged
insert / tunnel-interface vxlan0 vxlan-interface 211 ingress vni 211

insert / interface irb0 subinterface 211 ipv4 admin-state enable
insert / interface irb0 subinterface 211 ipv4 address 192.168.211.254/24 anycast-gw true
insert / interface irb0 subinterface 211 anycast-gw

insert / network-instance l2-irb2 type mac-vrf
insert / network-instance l2-irb2 vxlan-interface vxlan0.211
insert / network-instance l2-irb2 protocols bgp-evpn bgp-instance 1 vxlan-interface vxlan0.211
insert / network-instance l2-irb2 protocols bgp-evpn bgp-instance 1 evi 211
insert / network-instance l2-irb2 protocols bgp-vpn bgp-instance 1 route-target export-rt target:65000:211
insert / network-instance l2-irb2 protocols bgp-vpn bgp-instance 1 route-target import-rt target:65000:211

# IP-VRF to tie it together

insert / network-instance l3-irb type ip-vrf
insert / network-instance l3-irb interface irb0.210
insert / network-instance l3-irb interface irb0.211
```

Linux configuration:
```bash

ip link add link eth1 name eth1.211 type vlan id 211
ip link set dev eth1.211 up
ip addr add 192.168.211.2/24 dev eth1.211
ip route add 192.168.210.0/23 via 192.168.211.254

# ... repeat on other clients
```

</details>

Do some ping tests between the clients now! What works... and what doesn't!?

`client1` and `client2` can ping each other, `client2` and `client3` can ping each other... but not `client1` and `client3`!

Let's recap what happens in an asymmetric model IRB, how a packet is forwarded from `client1` to `client3`:
- `client1` sends the packet to `client3`'s IP. Since it's not in the same subnet, `client1` uses the configured static route, and sends the packet with `irb0.210`'s MAC address as the destination MAC
- `leaf1` receives the packet into the MAC-VRF `l2-irb1`. Since the destination MAC is `irb0.210`'s MAC, it is sent to the IRB interface for handling.
- `irb0.210` and `irb0.211` is in the same IP-VRF, so traffic can be routed between them. `irb0.211` is picked as the egress IRB. Now all we need to know is the MAC of the destination, to be able to send this packet to the correct egress VTEP. `irb0.211` generates an ARP request from `leaf1` and sends it to all other VTEPs that have the `l2-irb2` MAC-VRF.
- `leaf2` receives the ARP request to its `l2-irb2` MAC-VRF, and forwards it out to all its ports
- `client3` receives the ARP request, and generates a response. The response is sent back to `irb0.211`'s anycast gateway MAC address.
- `leaf2` receives the ARP response in MAC-VRF `l2-irb2` and forwards it to `irb0.211`, which has that same destination MAC address

So... `leaf2` eats the ARP response meant for `leaf1`, as both of their IRB's MAC addresses are the same. We need a solution for this!

Thankfully, there are some mechanisms that will help us here: unsolicited ARP learning and advertising learned ARP/ND entries in EVPN! Learn more about these [here](https://documentation.nokia.com/srlinux/25-10/books/advanced-solutions/evpn-vxlan-layer-3.html#irb-subinterface-considerations) and use what you learned to configure the missing knobs on the IRB.

<details>
<summary>Task 45.2 solution part 2</summary>

```
insert / interface irb0 subinterface 210 ipv4 arp learn-unsolicited true
insert / interface irb0 subinterface 210 ipv4 arp host-route populate dynamic
insert / interface irb0 subinterface 210 ipv4 arp evpn advertise dynamic

insert / interface irb0 subinterface 211 ipv4 arp learn-unsolicited true
insert / interface irb0 subinterface 211 ipv4 arp host-route populate dynamic
insert / interface irb0 subinterface 211 ipv4 arp evpn advertise dynamic
```

</details>

Pings between `client1` and `client2` should start working at this point!

> [!IMPORTANT]
> 
> You should save all your hard work at this point!
> It's best if you have all the configuration saved into the startup-config files, so you can return to this point in your network whenever you'd like to.
> It's also a good idea to save the CLI commands issued in the Linux clients, used for creating the VLAN interfaces. 
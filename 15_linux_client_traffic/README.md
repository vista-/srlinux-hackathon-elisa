# Extend the existing clab topology with end devices and enable the communication between `client1` and `client2` by using static routes
## Task 15.1: Extend our clab topology with end devices and enable the communication between `client1` and `client2`
By this point you should be accustomed to clab topology, nodes, links etc..
The goal of this task is to extend the clab topology with end devices. 
Add to the topology `client1` and `client2` (`kind: linux` and `image: ghcr.io/srl-labs/network-multitool`) and add related links for `leaf1` to `client1` and `leaf2` to `client2`.


```                                                                  
       ┌─────────┐                 ┌─────────┐
       │         │ethernet-1/30    │         │   
       │  leaf1  │─────────────────│  leaf2  │     
       │         │    ethernet-1/30│         │    
       └─────────┘                 └─────────┘       
ethernet-1/1│               ethernet-1/1│ 
            │                           │            
            │                           │            
        eth1│                       eth1│ 
       ┌─────────┐                 ┌─────────┐       
       │         │                 │         │       
       │ client1 │                 │ client2 │       
       │         │                 │         │       
       └─────────┘                 └─────────┘       


```

<details>
<summary>Task 15.1 solution</summary>
Containerlab topology:

```yaml
name: hackathon

topology:
  nodes:
    leaf1:
      kind: nokia_srlinux
      image: ghcr.io/nokia/srlinux:25.10.1
    leaf2:
      kind: nokia_srlinux
      image: ghcr.io/nokia/srlinux:25.10.1
    client1:
      kind: linux
      image: ghcr.io/srl-labs/network-multitool
    client2:
      kind: linux
      image: ghcr.io/srl-labs/network-multitool

  links:
    - endpoints: ["leaf1:ethernet-1/30", "leaf2:ethernet-1/30"]
    - endpoints: [ "leaf1:ethernet-1/1", "client1:eth1" ]
    - endpoints: [ "leaf2:ethernet-1/1", "client2:eth1" ]
```
</details>

## Task 15.2: configure `leaf1`, `leaf2`, `client1` and `client2` interfaces.
We already have the routed connectivity between `leaf1` and `leaf2` from the previous task. Now we need to further configure the interfaces from `leaf1` to `client1` respectivly `leaf2` to `client2`. Follow the same procedure as we learned in section 05 - srlinux interfaces.


The following network addressing schema will be used:


```
                    192.168.0.0/30                   
                                                     
       ┌─────────┐.1             .2┌─────────┐
       │         │ethernet-1/30    │         │
       │  leaf1  │─────────────────│  leaf2  │
       │         │    ethernet-1/30│         │
       └─────────┘                 └─────────┘       
ethernet-1/1│ 192.168.1.1   ethernet-1/1│ 192.168.2.1
            │                           │            
            │                           │            
        eth1│ 192.168.1.2           eth1│ 192.168.2.2
       ┌─────────┐                 ┌─────────┐       
       │         │                 │         │       
       │ client1 │                 │ client2 │       
       │         │                 │         │       
       └─────────┘                 └─────────┘       
```
<details>
<summary>Task 15.2 solution</summary>

## Configuration on `leaf1`: 

`leaf1` to `client1` interface configuration
```
insert / interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
insert / interface ethernet-1/1 subinterface 0 ipv4 address 192.168.1.1/24
```
Add the interface to the `default` routing instance:
```
insert / network-instance default interface ethernet-1/1.0
```
```
A:admin@leaf1# show network-instance default interfaces ethernet-1/1.0
===============================================================================================================================================================================================================================================================================================================
Net instance    : default
Interface       : ethernet-1/1.0
Type            : routed
Oper state      : up
Ip mtu          : 1500
  Prefix                                    Origin       Status
  ===============================================================================================
  192.168.1.1/24                            static       preferred, primary
===============================================================================================================================================================================================================================================================================================================
```

## Configuration on `leaf2`:

`leaf2` to `client1` interface configuration
```
insert / interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
insert / interface ethernet-1/1 subinterface 0 ipv4 address 192.168.2.1/24
```
Add the interface to the `default` routing instance:
```
insert / network-instance default interface ethernet-1/1.0
```
```
A:admin@leaf2# show network-instance default interfaces ethernet-1/1.0
===============================================================================================================================================================================================================================================================================================================
Net instance    : default
Interface       : ethernet-1/1.0
Type            : routed
Oper state      : up
Ip mtu          : 1500
  Prefix                                    Origin       Status
  ===============================================================================================
  192.168.2.1/24                            static       preferred, primary
===============================================================================================================================================================================================================================================================================================================
```

## Configuration on `client1`:
Add IP address to eth1
```
ip addr add 192.168.1.2/24 dev eth1
```

Now let's test the L3 connectivity between `client1` and `leaf1`
```
/ # ping 192.168.1.1
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=9.20 ms
```


## Configuration on `client2`:
Add IP address to eth1
```
ip addr add 192.168.2.2/24 dev eth1
```

## Testing
Let's run some pings:

`client1` to `leaf1`
```
/ # ping 192.168.1.1
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=3.26 ms
```


`client2` to `leaf2`
```
/ # ping 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=4.42 ms
```

`client1` to `client2`
Ping should fail, there are no static routes on the leafs and default routes on the clients. 
```
ping 192.168.2.2
```
</details>


## Task 15.3: Add static routes to `client1`, `client2`, `leaf1` and `leaf2`.

In order to achieve end-to-end connectivity between `client1` and `client2`, we need to make sure that both clients have a route to the other client subnet via their respective leaf uplink interface, and that each leaf can route to the other leaf's client subnet. We are going to achieve this by applying static routes both on the Linux clients, and on the SR Linux leaf switches as well.

### Static routes - [`follow this static route documentation in SR Linux`](https://documentation.nokia.com/srlinux/25-10/books/config-basics/network-instances.html#static-routes)
- In SR Linux, static routes are configured per network-instance and always point to a next-hop group. 

### What a static route is
- It matches an IPv4 or IPv6 prefix and belongs to a specific network-instance. 
- Each network-instance has its own static routes, RIB, and FIB, so the same prefix can be used in different VRFs independently. 
- A static route is created under:
/network-instance <name>/static-routes/route <prefix> 

Key parameters per route:
- prefix – IPv4 or IPv6
- admin-state – enable (default) or disable
- metric – IGP metric
- preference – preference vs. other routes
- next-hop-group – reference to a configured next-hop group  

> [!NOTE]
> A route is installed in the FIB when it has the best preference/metric and its next-hop group has resolvable next-hops.

### Next-hop groups
Every static route must reference a next-hop group. 

Supported next-hop types include: 
- IP-numbered next-hops (IPv4/IPv6, with resolve true|false)
- MPLS next-hops (some 7250 IXR platforms)
- GRE tunnel next-hops (some 7250 IXR platforms)
- Unnumbered IPv4 next-hops (via interface-ref to a subinterface)

<details>
<summary>Task 15.3 solution </summary>

## Adding static routes 

`leaf1` to `client2` subnet
```
insert / network-instance default next-hop-groups group leaf2 nexthop 1 ip-address 192.168.0.2
insert / network-instance default static-routes route 192.168.2.0/24 next-hop-group leaf2
```

`leaf2` to `client1`
```
insert / network-instance default next-hop-groups group leaf1 nexthop 1 ip-address 192.168.0.1
insert / network-instance default static-routes route 192.168.1.0/24 next-hop-group leaf1
```

`client1` route towards `client2`
```
ip ro add 192.168.2.0/24 via 192.168.1.1
```
`client2` route towards `client1`
```
ip ro add 192.168.1.0/24 via 192.168.2.1
```
</details>

## Testing

Check the routing table of `leaf1` and `leaf2` to see the static routes.
```
A:root@leaf1# show network-instance default route-table
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+---------------------------+---------------------------+---------------------------+---------------------------+
|                   Prefix                   |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |      Next-hop (Type)      |    Next-hop Interface     |  Backup Next-hop (Type)   | Backup Next-hop Interface |
|                                            |       |            |                      |          | Network  |         |            |                           |                           |                           |                           |
|                                            |       |            |                      |          | Instance |         |            |                           |                           |                           |                           |
+============================================+=======+============+======================+==========+==========+=========+============+===========================+===========================+===========================+===========================+
| 192.168.0.0/30                             | 1     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.0.1 (direct)      | ethernet-1/30.0           |                           |                           |
| 192.168.0.1/32                             | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.0.3/32                             | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.1.0/24                             | 3     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.1.1 (direct)      | ethernet-1/1.0            |                           |                           |
| 192.168.1.1/32                             | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.1.255/32                           | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.2.0/24                             | 0     | static     | static_route_mgr     | True     | default  | 1       | 5          | 192.168.0.0/30            | ethernet-1/30.0           |                           |                           |
|                                            |       |            |                      |          |          |         |            | (indirect/local)          |                           |                           |                           |
+--------------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+---------------------------+---------------------------+---------------------------+---------------------------+
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 7
IPv4 prefixes with active routes     : 7
IPv4 prefixes with active ECMP routes: 0
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

```
A:root@leaf2# show network-instance default route-table
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+---------------------------+---------------------------+---------------------------+---------------------------+
|                   Prefix                   |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |      Next-hop (Type)      |    Next-hop Interface     |  Backup Next-hop (Type)   | Backup Next-hop Interface |
|                                            |       |            |                      |          | Network  |         |            |                           |                           |                           |                           |
|                                            |       |            |                      |          | Instance |         |            |                           |                           |                           |                           |
+============================================+=======+============+======================+==========+==========+=========+============+===========================+===========================+===========================+===========================+
| 192.168.0.0/30                             | 1     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.0.2 (direct)      | ethernet-1/30.0           |                           |                           |
| 192.168.0.2/32                             | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.0.3/32                             | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.1.0/24                             | 0     | static     | static_route_mgr     | True     | default  | 1       | 5          | 192.168.0.0/30            | ethernet-1/30.0           |                           |                           |
|                                            |       |            |                      |          |          |         |            | (indirect/local)          |                           |                           |                           |
| 192.168.2.0/24                             | 3     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.2.1 (direct)      | ethernet-1/1.0            |                           |                           |
| 192.168.2.1/32                             | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
| 192.168.2.255/32                           | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None                      | None                      |                           |                           |
+--------------------------------------------+-------+------------+----------------------+----------+----------+---------+------------+---------------------------+---------------------------+---------------------------+---------------------------+
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 7
IPv4 prefixes with active routes     : 7
IPv4 prefixes with active ECMP routes: 0
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```
`client1` routing table
```
/ # ip ro
default via 172.20.20.1 dev eth0 
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.7 
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.2 
192.168.2.0/24 via 192.168.1.1 dev eth1 
```

`client2` routing table

```
/ # ip ro
default via 172.20.20.1 dev eth0 
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.6 
192.168.1.0/24 via 192.168.2.1 dev eth1 
192.168.2.0/24 dev eth1 proto kernel scope link src 192.168.2.2 
```

Ping from `client1` to `client2` 
```
/ # ping 192.168.2.2
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=62 time=0.883 ms
64 bytes from 192.168.2.2: icmp_seq=2 ttl=62 time=0.949 ms
```


# Task 17: Replace the static routes on `leaf1` and `leaf2` with BGP.
The goal of this task is to go through the basics of BGP, how to setup BGP and exchange routes. The topology remains the same as the one from the previous task (not needed to redeploy the topology, and always a good ideea to save your progress in startup configuration files). We will replace the static routes with eBGP and exchange `client1` and `client2` subnets between `leaf1` and `leaf2`.

As a reference for the BGP configuration on SRL, check our documentation: https://documentation.nokia.com/srlinux/25-10/books/routing-protocols/bgp.html

## BGP
In SR Linux, BGP is a per–network-instance routing protocol that supports both underlay and overlay use cases (classic IPv4/IPv6, EVPN, VRFs, PE‑CE, etc.). Configuration always follows the same basic pattern:

1. Where BGP is configured  
BGP is configured under a specific network instance:

```
/network-instance default protocols bgp ...
/network-instance tenant-2 protocols bgp ...
```
- default is typically the global routing table / underlay. 
- ip-vrf / tenant VRFs use their own BGP instance   
Each network instance using BGP has its own RIB/FIB context.

2. Minimal global BGP configuration  
At a minimum you must configure:
- Autonomous system number
- Router ID
- Address-family activation
- Peer group (even if it contains no additional configuration)
- Neighbors
- Import/export policy for eBGP

3. eBGP default policy behavior
By default, SR Linux does not import or export any routes on eBGP sessions (RFC 8212 behavior).   
You must:
- Either attach explicit import-policy/export-policy on peers/groups, as above; or
- Relax the default using ebgp-default-policy in the BGP context.

## BGP routing policies

BGP routing policies in SR Linux control which BGP routes are accepted (imported), modified, and advertised (exported), and how. They are standard SR Linux routing policies applied in a BGP-specific context.

### What is a routing policy in SR Linux?

A routing policy is an ordered list of statements plus an optional default action. Each statement:

- Has zero or more match conditions (for example: protocol, prefix-set, community, family).
- Has a base action (policy-result accept / reject / next-statement / next-policy).
- May include route-modifying actions (e.g., set MED, local-pref, communities, AS-path). 

### Evaluation:

- Statements are checked in order; the first matching statement determines the actions.
- If no statement matches, the policy’s default-action is used; if that is not set, a protocol/context-specific default applies. 


### Where are BGP policies applied?

BGP supports both import and export policies. 

You can apply policies at:

- BGP instance level
- BGP group (peer-group) level
- Neighbor level 

### Precedence:

- Neighbor-level policy overrides group and instance.
- Group-level policy overrides instance-level.
- If no policy at any level, default handling depends on iBGP vs eBGP 


In our case we will create a prefix set that is matching to our client prefix and will use a policy statment that is matching on protocol local and the created prefix.

```
    routing-policy {
        prefix-set clients {
            prefix 192.168.0.0/22 mask-length-range 24..24 {
            }
        }
        policy export-underlay {
            statement accept-clients {
                match {
                    protocol local
                    prefix {
                        prefix-set clients
                    }
                }
                action {
                    policy-result accept
                }
            }
        }
        policy import-underlay {
            statement accept-clients {
                match {
                    prefix {
                        prefix-set clients
                    }
                }
                action {
                    policy-result accept
                }
            }
        }
    }
```









## Configuring bgp between `leaf1` and `leaf2`.
 Use ASN65001 for `leaf1` and ASN65002 for `leaf2`.  
 Router ids on `leaf1` and `leaf2` should be 10.0.0.1 and 10.0.0.2.  
 Apply export/import policies in order to exchange `client1` and `client2` prefixes.  Try to put as much configuration as possible under the peer group level, which should be called `ebgp-underlay`.  

> [!TIP]
> To set up BGP routing, participating routers must have BGP enabled, and be assigned to an AS, and the neighbor (peer) relationships must be specified. A router typically belongs to only one AS.
> Int this section we configure the minimal configuration necessary to set up BGP in SR Linux. This includes the following:
> - Global BGP configuration, including specifying the Autonomous System Number (ASN) of the router, as well as the router ID.
> - BGP peer group configuration, which specifies settings that are applied to BGP neighbor routers in the peer group.
> - BGP neighbor configuration, which specifies the peer group to which each BGP neighbor belongs, as well as settings specific to the neighbor, including the AS to which the router is peered  
> - Atleast one address family must be enabled on the global BGP context.  

<details>
<summary>Task 17 solution</summary>

## BGP global configuration

`leaf1`
```
insert / network-instance default protocols bgp autonomous-system 65001
insert / network-instance default protocols bgp router-id 10.0.0.1
insert / network-instance default protocols bgp afi-safi ipv4-unicast admin-state enable
```
`leaf2`
```
insert / network-instance default protocols bgp autonomous-system 65002
insert / network-instance default protocols bgp router-id 10.0.0.2
insert / network-instance default protocols bgp afi-safi ipv4-unicast admin-state enable
```

## Configuring a BGP peer group
`leaf1`
```
insert / network-instance default protocols bgp group ebgp-underlay export-policy [ export-underlay ] first
insert / network-instance default protocols bgp group ebgp-underlay import-policy [ import-underlay ] first
```
`leaf2`
```
insert / network-instance default protocols bgp group ebgp-underlay export-policy [ export-underlay ] first
insert / network-instance default protocols bgp group ebgp-underlay import-policy [ import-underlay ] first
```

## Configuring BGP neighbors
`leaf1`
```
insert / network-instance default protocols bgp neighbor 192.168.0.2 peer-group ebgp-underlay
insert / network-instance default protocols bgp neighbor 192.168.0.2 peer-as 65002
```
`leaf2`
```
insert / network-instance default protocols bgp neighbor 192.168.0.1 peer-group ebgp-underlay
insert / network-instance default protocols bgp neighbor 192.168.0.1 peer-as 65001
```

### Checking the BGP session states - State should appear as `established`
```
A:root@leaf1# show network-instance default protocols bgp neighbor
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+---------------------+------------------------------+---------------------+-------+-----------+-----------------+-----------------+---------------+------------------------------+
|      Net-Inst       |             Peer             |        Group        | Flags |  Peer-AS  |      State      |     Uptime      |   AFI/SAFI    |        [Rx/Active/Tx]        |
+=====================+==============================+=====================+=======+===========+=================+=================+===============+==============================+
| default             | 192.168.0.2                  | ebgp-underlay       | S     | 65002     | established     | 0d:1h:42m:12s   | ipv4-unicast  | [1/1/1]                      |
+---------------------+------------------------------+---------------------+-------+-----------+-----------------+-----------------+---------------+------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
1 configured neighbors, 1 configured sessions are established, 0 disabled peers
0 dynamic peers
```

```
A:root@leaf1# show network-instance default protocols bgp neighbor
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
BGP neighbor summary for network-instance "default"
Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, - disabled, * slow
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+---------------------+------------------------------+---------------------+-------+-----------+-----------------+-----------------+---------------+------------------------------+
|      Net-Inst       |             Peer             |        Group        | Flags |  Peer-AS  |      State      |     Uptime      |   AFI/SAFI    |        [Rx/Active/Tx]        |
+=====================+==============================+=====================+=======+===========+=================+=================+===============+==============================+
| default             | 192.168.0.2                  | ebgp-underlay       | S     | 65002     | established     | 0d:1h:42m:12s   | ipv4-unicast  | [1/1/1]                      |
+---------------------+------------------------------+---------------------+-------+-----------+-----------------+-----------------+---------------+------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Summary:
1 configured neighbors, 1 configured sessions are established, 0 disabled peers
0 dynamic peers
```



## BGP peer import and export policies
For now, the client subnet info is passed directly via underlay
`leaf1`
```
insert / routing-policy prefix-set clients prefix 192.168.0.0/22 mask-length-range 24..24

insert / routing-policy policy export-underlay statement accept-clients first
insert / routing-policy policy export-underlay statement accept-clients match protocol local
insert / routing-policy policy export-underlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy export-underlay statement accept-clients action policy-result accept

insert / routing-policy policy import-underlay statement accept-clients first
insert / routing-policy policy import-underlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy import-underlay statement accept-clients action policy-result accept
```
`leaf2`
```
insert / routing-policy prefix-set clients prefix 192.168.0.0/22 mask-length-range 24..24

insert / routing-policy policy export-underlay statement accept-clients first
insert / routing-policy policy export-underlay statement accept-clients match protocol local
insert / routing-policy policy export-underlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy export-underlay statement accept-clients action policy-result accept

insert / routing-policy policy import-underlay statement accept-clients first
insert / routing-policy policy import-underlay statement accept-clients match prefix prefix-set clients
insert / routing-policy policy import-underlay statement accept-clients action policy-result accept
```
### Checking the IPv4 advertised routes by the neighbor
```
A:root@leaf1# show network-instance default protocols bgp neighbor 192.168.0.2 advertised-routes ipv4
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Peer        : 192.168.0.2, remote AS: 65002, local AS: 65001
Type        : static
Description : None
Group       : ebgp-underlay
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Origin codes: i=IGP, e=EGP, ?=incomplete
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|              Network                        Path-id                Next Hop             MED                               LocPref                             AsPath           Origin     |
+===========================================================================================================================================================================================+
| 192.168.1.0/24                      0                          192.168.0.1               -                                                               [65001]                  i       |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1 advertised BGP routes
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```
```
A:root@leaf2# show network-instance default protocols bgp neighbor 192.168.0.1 advertised-routes ipv4
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Peer        : 192.168.0.1, remote AS: 65001, local AS: 65002
Type        : static
Description : None
Group       : ebgp-underlay
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Origin codes: i=IGP, e=EGP, ?=incomplete
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|              Network                        Path-id                Next Hop             MED                               LocPref                             AsPath           Origin     |
+===========================================================================================================================================================================================+
| 192.168.2.0/24                      0                          192.168.0.2               -                                                               [65002]                  i       |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1 advertised BGP routes
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```

```
A:root@leaf1# show network-instance default protocols bgp routes ipv4 summary
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "default"
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+----------+---------------------------------------------+---------------------------------------------+----------+----------+--------------------------------------------------------------+
|  Status  |                   Network                   |                  Next Hop                   |   MED    | LocPref  |                           Path Val                           |
+==========+=============================================+=============================================+==========+==========+==============================================================+
| u*>      | 192.168.0.0/30                              | 0.0.0.0                                     |          |          |  i                                                           |
| u*>      | 192.168.1.0/24                              | 0.0.0.0                                     |          |          |  i                                                           |
| u*>      | 192.168.2.0/24                              | 192.168.0.2                                 |          |          |  ?                                                           |
| *        | 192.168.2.0/24                              | 192.168.0.2                                 |          |          |  i[65002]                                                    |
+----------+---------------------------------------------+---------------------------------------------+----------+----------+--------------------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
4 received BGP routes: 3 used, 4 valid, 0 stale
3 available destinations: 1 with ECMP multipaths
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```

```
A:root@leaf2# show network-instance default protocols bgp routes ipv4 summary
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Show report for the BGP route table of network-instance "default"
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Status codes: u=used, *=valid, >=best, x=stale, b=backup, w=unused-weight-only
Origin codes: i=IGP, e=EGP, ?=incomplete
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+----------+---------------------------------------------+---------------------------------------------+----------+----------+--------------------------------------------------------------+
|  Status  |                   Network                   |                  Next Hop                   |   MED    | LocPref  |                           Path Val                           |
+==========+=============================================+=============================================+==========+==========+==============================================================+
| u*>      | 192.168.0.0/30                              | 0.0.0.0                                     |          |          |  i                                                           |
| u*>      | 192.168.1.0/24                              | 192.168.0.1                                 |          |          |  ?                                                           |
| *        | 192.168.1.0/24                              | 192.168.0.1                                 |          |          |  i[65001]                                                    |
| u*>      | 192.168.2.0/24                              | 0.0.0.0                                     |          |          |  i                                                           |
+----------+---------------------------------------------+---------------------------------------------+----------+----------+--------------------------------------------------------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
4 received BGP routes: 3 used, 4 valid, 0 stale
3 available destinations: 1 with ECMP multipaths
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```
### Checking the routing table
```
A:root@leaf1# show network-instance default route-table
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
|          Prefix          |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |    Next-hop    |    Next-hop    |  Backup Next-  |  Backup Next-  |
|                          |       |            |                      |          | Network  |         |            |     (Type)     |   Interface    |   hop (Type)   | hop Interface  |
|                          |       |            |                      |          | Instance |         |            |                |                |                |                |
+==========================+=======+============+======================+==========+==========+=========+============+================+================+================+================+
| 192.168.0.0/30           | 1     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.0.1    | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       | 1/30.0         |                |                |
| 192.168.0.1/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.0.3/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.1.0/24           | 3     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.1.1    | ethernet-1/1.0 |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       |                |                |                |
| 192.168.1.1/32           | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.1.255/32         | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.2.0/24           | 0     | static     | static_route_mgr     | True     | default  | 1       | 5          | 192.168.0.0/30 | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (indirect/loca | 1/30.0         |                |                |
|                          |       |            |                      |          |          |         |            | l)             |                |                |                |
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 7
IPv4 prefixes with active routes     : 7
IPv4 prefixes with active ECMP routes: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```

```
A:root@leaf2# show network-instance default route-table
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
|          Prefix          |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |    Next-hop    |    Next-hop    |  Backup Next-  |  Backup Next-  |
|                          |       |            |                      |          | Network  |         |            |     (Type)     |   Interface    |   hop (Type)   | hop Interface  |
|                          |       |            |                      |          | Instance |         |            |                |                |                |                |
+==========================+=======+============+======================+==========+==========+=========+============+================+================+================+================+
| 192.168.0.0/30           | 1     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.0.2    | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       | 1/30.0         |                |                |
| 192.168.0.2/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.0.3/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.1.0/24           | 0     | static     | static_route_mgr     | True     | default  | 1       | 5          | 192.168.0.0/30 | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (indirect/loca | 1/30.0         |                |                |
|                          |       |            |                      |          |          |         |            | l)             |                |                |                |
| 192.168.2.0/24           | 3     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.2.1    | ethernet-1/1.0 |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       |                |                |                |
| 192.168.2.1/32           | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.2.255/32         | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 7
IPv4 prefixes with active routes     : 7
IPv4 prefixes with active ECMP routes: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```
  
</details>  

> [!TIP]
> At this point the BGP IPv4 routes are not installed in the routing table, the static route is still present having a better (5) prefference than BGP (170) routes.   
> We can just disable the static routes, in order to have the BGP routes installed into the routing table.

<details>

<summary>Disable static route solution </summary>  

`leaf1`  

```
insert / network-instance default static-routes route 192.168.2.0/24 admin-state disable
```
`leaf2`
```
insert / network-instance default static-routes route 192.168.1.0/24 admin-state disable
```

### Checking the routing table, expecting to see the BGP advertised routes installed in the routing table
```
A:root@leaf1# show network-instance default route-table
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
|          Prefix          |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |    Next-hop    |    Next-hop    |  Backup Next-  |  Backup Next-  |
|                          |       |            |                      |          | Network  |         |            |     (Type)     |   Interface    |   hop (Type)   | hop Interface  |
|                          |       |            |                      |          | Instance |         |            |                |                |                |                |
+==========================+=======+============+======================+==========+==========+=========+============+================+================+================+================+
| 192.168.0.0/30           | 1     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.0.1    | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       | 1/30.0         |                |                |
| 192.168.0.1/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.0.3/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.1.0/24           | 3     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.1.1    | ethernet-1/1.0 |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       |                |                |                |
| 192.168.1.1/32           | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.1.255/32         | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.2.0/24           | 0     | bgp        | bgp_mgr              | True     | default  | 0       | 170        | 192.168.0.0/30 | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (indirect/loca | 1/30.0         |                |                |
|                          |       |            |                      |          |          |         |            | l)             |                |                |                |
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 7
IPv4 prefixes with active routes     : 7
IPv4 prefixes with active ECMP routes: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```
```
A:root@leaf2# show network-instance default route-table
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 unicast route table of network instance default
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
|          Prefix          |  ID   | Route Type |     Route Owner      |  Active  |  Origin  | Metric  |    Pref    |    Next-hop    |    Next-hop    |  Backup Next-  |  Backup Next-  |
|                          |       |            |                      |          | Network  |         |            |     (Type)     |   Interface    |   hop (Type)   | hop Interface  |
|                          |       |            |                      |          | Instance |         |            |                |                |                |                |
+==========================+=======+============+======================+==========+==========+=========+============+================+================+================+================+
| 192.168.0.0/30           | 1     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.0.2    | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       | 1/30.0         |                |                |
| 192.168.0.2/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.0.3/32           | 1     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.1.0/24           | 0     | bgp        | bgp_mgr              | True     | default  | 0       | 170        | 192.168.0.0/30 | ethernet-      |                |                |
|                          |       |            |                      |          |          |         |            | (indirect/loca | 1/30.0         |                |                |
|                          |       |            |                      |          |          |         |            | l)             |                |                |                |
| 192.168.2.0/24           | 3     | local      | net_inst_mgr         | True     | default  | 0       | 0          | 192.168.2.1    | ethernet-1/1.0 |                |                |
|                          |       |            |                      |          |          |         |            | (direct)       |                |                |                |
| 192.168.2.1/32           | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
| 192.168.2.255/32         | 3     | host       | net_inst_mgr         | True     | default  | 0       | 0          | None           | None           |                |                |
+--------------------------+-------+------------+----------------------+----------+----------+---------+------------+----------------+----------------+----------------+----------------+
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 routes total                    : 7
IPv4 prefixes with active routes     : 7
IPv4 prefixes with active ECMP routes: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```


### Clients should be able to ping from one side to the other
Ping from `client1` to `client2`
```
/ # ping 192.168.2.2
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=62 time=1.13 ms
64 bytes from 192.168.2.2: icmp_seq=2 ttl=62 time=0.917 ms
```

</details>
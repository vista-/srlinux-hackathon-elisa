# SR Linux interfaces and network-instances

## Task 05.1: Creating your first SR Linux network lab

To begin this task, we advise you to create a topology directory in the root of the Hackathon repository and use this as your working directory for the rest of the Hackathon. The tasks build upon each other, and require the topology and configuration work you have performed in earlier tasks to work correctly. 

For this workshop, we will use SR Linux as our main network OS of choice, which can be freely pulled from the GitHub Container Repository (GHCR), and let's use the latest release of SR Linux (at the time of writing), 25.10.1. To find the image URL, take a look on the SR Linux kind's [`documentation`](https://containerlab.dev/manual/kinds/srl/) on the Containerlab website!

The Containerlab documentation is always a good starting point for creating new topologies, especially if you want to use a network OS that you haven't worked with before in Containerlab.

The hackathon topology will be very creatively called "hackathon"! The topology consists of two nodes running SR Linux,  called `leaf1` and `leaf2`, and should be connected over their `ethernet-1/30` interfaces.

A simplified diagram of the topology can be seen below:

```
┌─────────┐                 ┌─────────┐
│         │ethernet-1/30    │         │
│  leaf1  │─────────────────│  leaf2  │
│         │    ethernet-1/30│         │
└─────────┘                 └─────────┘ 
```

Just as a quick note: in Containerlab topologies, it is possible to use the network OS's own interface naming when defining links, with some exceptions - luckily, both containerized SR Linux and SR OS (SR-SIM) fully support defining links using the NOSes own interface terminology.

**Let's go!** Your goal is to first deploy the topology, connect to `leaf1`, check out the version output, and to verify that `leaf1` is connected to `leaf2` using `ethernet-1/30` via LLDP.

To connect to SR Linux nodes, we will _not_ use Docker exec to run a command on the node, but rather we will just simply SSH to it.

Containerlab will pre-configure most nodes with enough startup configuration to number the management interfaces (as mentioned before), and will configure pre-defined user credentials to enable you to connect via SSH or other management interfaces, like gNMI/gRPC.  
The Containerlab documentation of the kind will have these default credentials documented.

<details>
<summary>Task 05.1 solution</summary>
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

  links:
    - endpoints: ["leaf1:ethernet-1/30", "leaf2:ethernet-1/30"]
```

Version and LLDP check:
```
$ ssh admin@clab-hackathon-leaf1
Warning: Permanently added 'clab-hackathon-leaf1' (ED25519) to the list of known hosts.
................................................................
:                  Welcome to Nokia SR Linux!                  :
:              Open Network OS for the NetOps era.             :
:                                                              :
:    This is a freely distributed official container image.    :
:                      Use it - Share it                       :
:                                                              :
: Get started: https://learn.srlinux.dev                       :
: Container:   https://go.srlinux.dev/container-image          :
: Docs:        https://doc.srlinux.dev/25-10                   :
: Rel. notes:  https://doc.srlinux.dev/rn25-10-1               :
: YANG:        https://yang.srlinux.dev/v25.10.1               :
: Discord:     https://go.srlinux.dev/discord                  :
: Contact:     https://go.srlinux.dev/contact-sales            :
................................................................

Loading environment configuration file(s): ['/etc/opt/srlinux/srlinux.rc', '/home/admin/.srlinuxrc']
Welcome to the Nokia SR Linux CLI.


--{ running }--[  ]--
A:admin@leaf1# show version
---------------------------------------------------
Hostname             : leaf1
Chassis Type         : 7220 IXR-D2L
Part Number          : Sim Part No.
Serial Number        : Sim Serial No.
System HW MAC Address: 1A:1E:00:FF:00:00
OS                   : SR Linux
Software Version     : v25.10.1
Build Number         : 399-g90c1dbe35ef
Architecture         : aarch64
Last Booted          : 2025-12-03T16:26:38.033Z
Total Memory         : 16341768 kB
Free Memory          : 8478576 kB
---------------------------------------------------

--{ running }--[  ]--
A:admin@leaf1# show system lldp neighbor
...
| ethernet-1/30 | 1A:3E:01:FF:00:00 | leaf2                | 1A:3E:01:FF:00:00   | ...
...
```
</details>


## Task 05.2: configure `leaf1` to `leaf2` L3 connectivity
Now that we have the clab topology in place, we will configure the interfaces on `leaf1` and `leaf2` in order to have L3 connectivity between them.

What is a routed subinterface?
A routed subinterface:
- Is a logical channel under a parent interface (Ethernet, LAG, loopback, system, mgmt, or IRB).   
- Has type routed in the interface data model.  
- Can be bound to network-instances of type default, ip-vrf, or mgmt.  
- On the wire, routed subinterfaces carry Layer‑3 traffic and support IPv4/IPv6 configuration (addresses, MTU, routing protocols, etc.).  

Where can routed subinterfaces be used?  
- Ethernet and LAG ports – e.g. ethernet-1/1.0, lag1.100.  
- Loopbacks – loN.0.   
- System and mgmt – system0.0, mgmt0.0.   
- IRB interfaces – irbN.x subinterfaces are always type routed.   

### Configure the subinterfaces on `leaf1 ethernet-1/30` and on `leaf2 ethernet-1/30` 
Check the  [`Learn SR Linux - Interfaces`](https://learn.srlinux.dev/get-started/interface/) to get started with the interface configuration.

The following network addressing schema will be used:


```
                
                   192.168.0.0/30                                                                       
      ┌─────────┐.1             .2┌─────────┐
      │         │ethernet-1/30    │         │
      │  leaf1  │─────────────────│  leaf2  │
      │         │    ethernet-1/30│         │
      └─────────┘                 └─────────┘        

```
<details>
<summary>Task 05.2 solution part 1</summary>

### Configuration on `leaf1`: 
`leaf1` to `leaf2` interface configuration
```
insert / interface ethernet-1/30 subinterface 0 ipv4 admin-state enable
insert / interface ethernet-1/30 subinterface 0 ipv4 address 192.168.0.1/30
```
```
A:root@leaf1# show interface ethernet-1/30
==========================================================================================================================================================================================================================================================================================
ethernet-1/30 is up, speed 25G, type None
  ethernet-1/30.0 is up
    Network-instances:
    Encapsulation   : null
    Type            : routed
    IPv4 addr    : 192.168.0.1/30 (static, None)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
==========================================================================================================================================================================================================================================================================================

--{ running }--[  ]--
A:root@leaf1# show interface ethernet-1/30.0
==========================================================================================================================================================================================================================================================================================
  ethernet-1/30.0 is up
    Network-instances:
    Encapsulation   : null
    Type            : routed
    IPv4 addr    : 192.168.0.1/30 (static, None)
==========================================================================================================================================================================================================================================================================================
```






### Configuration on `leaf2`:
`leaf2` to `leaf1` interface configuration
```
insert / interface ethernet-1/30 subinterface 0 ipv4 admin-state enable
insert / interface ethernet-1/30 subinterface 0 ipv4 address 192.168.0.2/30
```



```
A:root@leaf2# show interface ethernet-1/30
==========================================================================================================================================================================================================================================================================================
ethernet-1/30 is up, speed 25G, type None
  ethernet-1/30.0 is up
    Network-instances:
    Encapsulation   : null
    Type            : routed
    IPv4 addr    : 192.168.0.2/30 (static, None)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
==========================================================================================================================================================================================================================================================================================

--{ running }--[  ]--
A:root@leaf2# show interface ethernet-1/30.0
==========================================================================================================================================================================================================================================================================================
  ethernet-1/30.0 is up
    Network-instances:
    Encapsulation   : null
    Type            : routed
    IPv4 addr    : 192.168.0.2/30 (static, None)
==========================================================================================================================================================================================================================================================================================

```

### Testing `leaf1` to `leaf2` l3 connectivity:
```
A:root@leaf1# ping 192.168.0.2
Using network instance <base>
ping: connect: Network is unreachable
```


</details>




> [!NOTE]
> Ping is failing, and this is an expected result. By default the router is trying to ping from `<base>` routing instance, and the newly created subinterfaces are not associated to any network routing instance (virtual router/VRF)  
> Subinterfaces `ethernet-1/30.0` have to be added to `default` network instance.  
> After this step, ping between `leaf1` and `leaf2` should work.  

### Network instances

On the SR Linux device, you can configure one or more virtual routing instances, known as network instances. Each network instance has its own interfaces, its own protocol instances, its own route table, and its own FIB.

When a packet arrives on a subinterface associated with a network instance, it is forwarded according to the FIB of that network instance. Transit packets are normally forwarded out another subinterface of the network instance.

SR Linux supports the following types of network instances:

- default
- ip-vrf
- mac-vrf

The initial startup configuration for SR Linux has a single default network instance. By default, there are no ip-vrf or mac-vrf network instances; these must be created by explicit configuration.  
The ip-vrf network instances are the building blocks of Layer 3 IP VPN services and Layer 3 EVPN services, and mac-vrf network instances are the building blocks of Layer2 EVPN services.

Within a network instance, you can configure BGP, OSPF, and IS-IS protocol options that apply only to that network instance. 

<details>
<summary>Task 05.2 solution part 2</summary>

### Add subinterface `ethernet-1/30.0` to `default` network instance on `leaf1`: 

```
insert / network-instance default interface ethernet-1/30.0
```
```
A:admin@leaf1# show network-instance default interfaces ethernet-1/30.0
===============================================================================================================================================================================================================================================================================================================
Net instance    : default
Interface       : ethernet-1/30.0
Type            : routed
Oper state      : up
Ip mtu          : 1500
  Prefix                                    Origin       Status
  ===============================================================================================
  192.168.0.1/30                            static       preferred, primary
===============================================================================================================================================================================================================================================================================================================
```

### Add subinterface `ethernet-1/30.0` to `default` network instance on `leaf2`: 
Add the interfaces to the `default` routing instance:
```
insert / network-instance default interface ethernet-1/30.0
```
```
A:admin@leaf2# show network-instance default interfaces ethernet-1/30.0
===============================================================================================================================================================================================================================================================================================================
Net instance    : default
Interface       : ethernet-1/30.0
Type            : routed
Oper state      : up
Ip mtu          : 1500
  Prefix                                    Origin       Status
  ===============================================================================================
  192.168.0.2/30                            static       tentative
===============================================================================================================================================================================================================================================================================================================
```

### Testing `leaf1` to `leaf2` l3 connectivity:
```
A:admin@leaf1# ping network-instance default 192.168.0.2
Using network instance default
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=6.29 ms
```

</details>

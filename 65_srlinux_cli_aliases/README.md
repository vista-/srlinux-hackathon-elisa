## Section 65: SR Linux CLI deep-dive

In most network OSes, `show` commands give you an idea, as a human operator, what is going on inside a switch.
    
```
--{ running }--[  ]--
A:admin@leaf1# show interface ethernet-1/1.200 detail
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
================================================================================================================================================================================================================================================================================================================
  Subinterface: ethernet-1/1.200
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Description     : <None>
  Network-instance: routed
  Type            : routed
  Oper state      : up
  Down reason     : N/A
  Last change     : 41m42s ago
  Encapsulation   : vlan-id 200
  IP MTU          : 1500
  Last stats clear: never
  IPv4 addr    : 192.168.201.1/24 (static, preferred, primary)
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ARP/ND summary for ethernet-1/1.200
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  IPv4 ARP entries : 0 static, 1 dynamic
  IPv6 ND  entries : 0 static, 0 dynamic
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Traffic statistics for ethernet-1/1.200
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Statistics        Rx    Tx 
  Packets             13     3  
  Octets              1142   306
  Discarded packets   10     0  
  Forwarded packets   3      3  
  Forwarded octets    306    306
  CPM packets         -      0  
  CPM octets          -      0  
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv4 Traffic statistics for ethernet-1/1.200
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
IPv6 Traffic statistics for ethernet-1/1.200
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
================================================================================================================================================================================================================================================================================================================
```   
    
But what we are really interested in right now is the pure _state_ of it. With SR Linux, it's easy to view the data model of the switch with `info`.
    
We already talked about `info` earlier, when we used to look at the configuration of the switch. The `running` and `candidate` configuration is also part of the data model of the switch, but it's specifically part of the data model that we can modify. What about the parts of the data model that are read-only?
    
There's a third "mode", or more properly, datastore, in SR Linux called `state`, which exposes those read-only parameters to us!

```
--{ + running }--[  ]--
A:leaf1# enter state

--{ + state }--[  ]--
A:leaf1# info interface ethernet-1/1 statistics
    in-packets 23
    in-octets 1798
    in-unicast-packets 5
    in-broadcast-packets 0
    in-multicast-packets 7
    in-discarded-packets 18
    in-error-packets 0
    in-fcs-error-packets 0
    out-packets 97
    out-octets 16339
    out-mirror-octets 0
    out-unicast-packets 4
    out-broadcast-packets 6
    out-multicast-packets 87
    out-discarded-packets 0
    out-error-packets 0
    out-mirror-packets 0
    carrier-transitions 1
```
    
Instead of doing `enter state`, we can ask our `info` command to fetch information from a specific datastore with the `from <datastore>` keyword. Let's try it out after jumping back into `running` mode!

```
--{ running }--[  ]--
A:leaf1# info from state interface ethernet-1/31 statistics                         
    in-packets 458
    in-octets 45092
    in-unicast-packets 358
    in-broadcast-packets 5
    in-multicast-packets 88
    in-discarded-packets 7
    in-error-packets 0
    in-fcs-error-packets 0
    out-packets 404
    out-octets 42416
    out-mirror-octets 0
    out-unicast-packets 309
    out-broadcast-packets 6
    out-multicast-packets 89
    out-discarded-packets 0
    out-error-packets 0
    out-mirror-packets 0
    carrier-transitions 0
```

Here are all the counters we were looking for, presented in a structured manner.
Don't believe me that it's structured data? Try to run `info from state interface ethernet-1/1.200`!

The SR Linux CLI also has more tricks up its sleeve! We have what are called filters at our disposal. Sort of like a `| include` (or `| match`), but so much more powerful! We can leverage the fact that the data shown here is structured data, and the SR Linux CLI can actually interpret it.

## Task 65.1: Formatting in CLI

One of these filters is called `as`. It allows you to specify the output format.

Your task is to output the interface counters of `ethernet-1/1` in YAML format!

<details>
<summary>Task 65.1 solution</summary>

```
--{ running }--[  ]--
A:admin@leaf1# info from state interface ethernet-1/1 statistics | as yaml
---
in-packets: 25
in-octets: 1946
in-unicast-packets: 5
in-broadcast-packets: 0
in-multicast-packets: 7
in-discarded-packets: 20
in-error-packets: 0
in-fcs-error-packets: 0
out-packets: 109
out-octets: 18487
out-mirror-octets: 0
out-unicast-packets: 4
out-broadcast-packets: 6
out-multicast-packets: 99
out-discarded-packets: 0
out-error-packets: 0
out-mirror-packets: 0
carrier-transitions: 1
```

</details>

## Task 65.2: Filtering fields

Let's say that we are only interested in certain parts of the output. Since we are working with structured data, this is as easy as `filtering` for only the `fields` we want.

Your next goal is to output only the in and out bytes counters of `ethernet-1/1`, as a table!

<details>
<summary>Task 65.2 solution</summary>

```
--{ running }--[  ]--
A:admin@leaf1# info from state interface ethernet-1/1 statistics | filter fields in-octets out-octets | as table
+---------------------+----------------------+----------------------+
|      Interface      |      In-octets       |      Out-octets      |
+=====================+======================+======================+
| ethernet-1/1        |                 2020 |                23320 |
+---------------------+----------------------+----------------------+
```

</details>

## Task 65.3: Wildcards

We have been showing the output for a single interface, so far. What happens if you replace the interface index with a wildcard character? You can use curly braces to specify specific items (comma-separated), ranges (marked with ..) or an asterisk to indicate a wildcard.

When dealing with a range of interfaces or other groups of items, sometimes you end up with entries that have no output associated with them. For example, in case of interfaces, if it was never turned on, or even defined in the configuration, it will naturally not have any statistics. We can use the `non-zero` filter to get rid of these extraneous outputs.

Show only the non-zero in and out bytes counters of _all interfaces_, as a table, as an exercise!

<details>
<summary>Task 65.3 solution</summary>

```
--{ running }--[  ]--
A:admin@leaf1# info from state interface ethernet-1/* statistics | filter non-zero fields in-octets out-octets  | as table
+---------------------+----------------------+----------------------+
|      Interface      |      In-octets       |      Out-octets      |
+=====================+======================+======================+
| ethernet-1/1        |                 2020 |                24931 |
| ethernet-1/30       |                  560 |                    0 |
| ethernet-1/31       |                68051 |                64912 |
| ethernet-1/32       |                64060 |                65113 |
+---------------------+----------------------+----------------------+
```

</details>

## Task 65.4: CLI utilities

Finally, let's take a look at what happens when we don't suffix a command, but prefix it to help us get better output!

By pressing the question mark button twice, we can see all the utility commands at our disposal in the CLI. For example, `tree` shows all the hierarchy under a specific context. This allows you to discover the YANG model from within the CLI.

```
--{ running }--[  ]--
A:admin@leaf1# tree from state interface ethernet statistics
statistics
+-- in-mac-pause-frames
+-- in-oversize-frames
+-- in-undersize-frames
+-- in-jabber-frames
+-- in-fragment-frames
+-- in-crc-error-frames
+-- out-mac-pause-frames
+-- in-64b-frames
+-- in-65b-to-127b-frames
+-- in-128b-to-255b-frames
+-- in-256b-to-511b-frames
+-- in-512b-to-1023b-frames
+-- in-1024b-to-1518b-frames
+-- in-1519b-or-longer-frames
+-- out-64b-frames
+-- out-65b-to-127b-frames
+-- out-128b-to-255b-frames
+-- out-256b-to-511b-frames
+-- out-512b-to-1023b-frames
+-- out-1024b-to-1518b-frames
+-- out-1519b-or-longer-frames
+-- last-clear
```

Use one of the utilities to give you a constantly refreshing table of interface in and out packet counters!

Start a ping and see the counters increase!

<details>
<summary>Task 65.4 solution</summary>

```
--{ running }--[  ]--
A:admin@leaf1# watch info from state interface ethernet-1/* statistics | filter non-zero fields in-packets out-packets | as table

Every 2.0s: info from state interface ethernet-1/* statistics | filter fields in-packets out-packets non-zero | as table
+---------------------+----------------------+----------------------+
|      Interface      |      In-packets      |     Out-packets      |
+=====================+======================+======================+
| ethernet-1/1        |                   28 |                  165 |
| ethernet-1/30       |                    8 |                    0 |
| ethernet-1/31       |                  794 |                  716 |
| ethernet-1/32       |                  739 |                  730 |
+---------------------+----------------------+----------------------+
```

</details>

## Task 65.5: CLI aliases
With SR Linux, users have the flexibility to create aliases for long, complex CLI commands, or reformat them in a more familiar way. As a shortcut for entering commands, you can configure CLI command aliases using the `environment alias` command. An alias can include one or more CLI command keywords and arguments.  


In this task, you will learn how to implement aliases in SR Linux through hands-on examples. By following these examples, you will gain practical experience. At the end of the lab, there is an exercise to test what you have learned.


* Basic alias examples
* Aliases with parameter


```
       Create alias command        Alias           SR Linux command 
                                                                    
                 │                   │                     │        
                 │                   │                     │        
                 │                   │                     │        
                                                                    
A:leaf1# environment alias        configure        "enter candidate"
```
Follow the documentation about [SR Linux configuring aliases](https://documentation.nokia.com/srlinux/24-10/books/system-mgmt/cli-interface.html#configuring-cli-command-aliases)

### Basic Commands

We will start with a very basic example. In order to create an alias, you will need to use the `environment alias` command followed by the alias you want to create and the existing command you want to replace.

#### Configure an alias for the following command, name the alias `configure`
```sh
A:root@leaf1# enter candidate
```
<details> 
<summary>Solution </summary>

```sh 
environment alias "configure" "enter candidate"
```
</details>

### Advanced commands using parameters
Make an alias command for the bellow command and try to use arguments for the interface name.
```sh
info from state interface * statistics | filter fields in-error-packets out-error-packets | as table
```

<details> 
<summary>Solution </summary>

```sh 
A:root@leaf1# environment alias "show statistics" "info from state interface {interface} statistics | filter fields in-error-packets out-error-packets | as table"
```
</details>  

<br>
<br>

> [!TIP]  
> If you want to reuse, do not forget to save your aliases
> Aliases are ephemeral unless saved to the environment file. Aliases are saved in a separate file from the configuration file.

_Saving environment settings:_

```sh
environment save home

Saved configuration to /admin/.srlinuxrc
--{ + running }--[  ]--
```

# Section 50: Streaming Telemetry, gNMIc

By the end of this workshop, you will build a streaming telemetry-based monitoring system for your virtual data center!

We're going to be using the following building blocks to do so:
- **gNMIc**  
is an open-source gNMI client, which allows you to collect information via streaming telemetry from a gRPC-enabled network node. For example, in this topology, SR Linux works with gNMI out of the box, and requires no additional configuration.
- **Prometheus**  
is a very popular time-series database (TSDB). We will use it to query and store the data collected by gNMIc.
- **Grafana**  
is the one of the most popular tools for data visualization. It is used across various industries, and allows us to visualize the data that is stored in various data sources, like Prometheus.  

## Task 50.1: gNMIc installation

[gNMIc](https://gnmic.openconfig.net/) can be used as a CLI tool to create telemetry subscriptions. Let's install it! Thankfully, the website gives us a one-liner install command, no complicated steps here.

<details>
<summary>Task 50.1 solution</summary>

```shell
$ bash -c "$(curl -sL https://get-gnmic.openconfig.net)"
Downloading https://github.com/openconfig/gnmic/releases/download/v0.42.1/gnmic_0.42.1_linux_aarch64.tar.gz
Preparing to install gnmic 0.42.1 into /usr/local/bin
gnmic installed into /usr/local/bin/gnmic
version : 0.42.1
 commit : 6b35566f
   date : 2025-10-20T17:04:55Z
 gitURL : https://github.com/openconfig/gnmic
   docs : https://gnmic.openconfig.net
```

</details>

## Task 50.2: gNMIc usage

Now to try out gNMIc! Use the `--help` output to put in all the right command line flags. Try to fetch the statistics of ethernet-1/1 on leaf1.

You'll need, at minimum:
- hostname/IP
- username
- password
- encoding (use `json_ietf`)
- gNMIc command (use `get`)
- YANG path

A very handy tool when dealing with YANG in SR Linux is the [SR Linux YANG Explorer](https://yang.srlinux.dev/), it's really handy for finding not only YANG paths, but also for navigating the CLI!

<details>
<summary>Task 50.2 solution</summary>

```json
$ gnmic -a clab-hackathon-leaf1 -u admin -p NokiaSrl1! --skip-verify -e json_ietf get --path "/interface[name=ethernet-1/1]/statistics"
[
  {
    "source": "clab-hackathon-leaf1",
    "timestamp": 1765310986253298661,
    "time": "2025-12-09T21:09:46.253298661+01:00",
    "updates": [
      {
        "Path": "srl_nokia-interfaces:interface[name=ethernet-1/1]/statistics",
        "values": {
          "srl_nokia-interfaces:interface/statistics": {
            "carrier-transitions": "1",
            "in-broadcast-packets": "1",
            "in-discarded-packets": "45",
            "in-error-packets": "0",
            "in-fcs-error-packets": "0",
            "in-multicast-packets": "10",
            "in-octets": "393522",
            "in-packets": "3937",
            "in-unicast-packets": "3891",
            "out-broadcast-packets": "24",
            "out-discarded-packets": "0",
            "out-error-packets": "0",
            "out-mirror-octets": "0",
            "out-mirror-packets": "0",
            "out-multicast-packets": "543",
            "out-octets": "108083",
            "out-packets": "698",
            "out-unicast-packets": "131"
          }
        }
      }
    ]
  }
]
```
</details>

Now, the CLI command is really, really long... So instead, we should create the gNMIc [client configuration](https://gnmic.openconfig.net/user_guide/configuration_intro/) in `~/.gnmic.yaml` with the username, password, encoding and skip-verify set!

<details>
<summary>Task 50.2 gNMIC config solution</summary>

```
username: admin
password: NokiaSrl1!
port: 57400
timeout: 5s
skip-verify: true
encoding: json_ietf
```
</details>

Now we can use gNMIc with much less configuration:

```json
$ gnmic -a clab-hackathon-leaf1 get --path "/interface[name=ethernet-1/1]/statistics/in-octets"
[
  {
    "source": "clab-hackathon-leaf1",
    ...
  }
]
```

## Task 50.3 Streaming Telemetry basics

If we switch from `get` to `subscribe`, we will see how streaming telemetry works!

What do you observe when you let this command keep running for a while longer? That's right, the counters keep getting updated!

Try to subscribe to another gNMI path: inside the statistics, let's focus on the output packet counter, but for _every_ interface!  
gNMI paths are not fixed strings, but what are called an XPath expression. The keys used in these expressions can be partially or totally replaced with wildcards (*). What do you see when you subscribe to this path?

<details>
<summary>Task 50.3 part 1 solution</summary>

```json
$ gnmic -a clab-hackathon-leaf1 subscribe --path "/interface[name=ethernet-*]/statistics/out-packets"
...
```

Multiple interfaces show up in the response, in multiple responses (each response has its own timestamp). At the end of the first subscription update, a separate message is sent to let the server know that all data has been transferred. After this, every 5 seconds, new updates are sent, even for interfaces where the counter did not increase in the last 5 seconds.
</details>

We can also monitor multiple devices at once: **try to monitor both `leaf1` and `spine1` at the same time, with the same command.**

<details>
<summary>Task 50.2 part 2 solution</summary>

```json
$ gnmic -a clab-hackathon-leaf1,clab-hackathon-spine1 subscribe --path "/interface[name=ethernet-*]/statistics/out-packets"
{
  "source": "clab-hackathon-leaf1",
...
}
{
  "source": "clab-hackathon-spine1",
...
}
```
</details>

Once you successfully do this, adjust the path to instead monitor the operational state of every interface.  
Do you notice anything different in how this subscription works?

Try to set admin-state to disable for the uplink between `leaf1` and `spine1`, and flap it back to enabled as soon as possible.

<details>
<summary>Task 50.2 part 3 solution</summary>

```json
$ gnmic -a clab-hackathon-leaf1,clab-hackathon-spine1 subscribe --path "/interface[name=ethernet-*]/oper-state"
# Output all interfaces' on both leaf1 and spine1, but does not update afterwards

# on leaf1: set interface ethernet-1/31 admin-state disable
# commit
{
  "source": "clab-hackathon-leaf1",
  "subscription-name": "default-1765311864",
  "timestamp": 1765312051566156674,
  "time": "2025-12-09T21:27:31.566156674+01:00",
  "updates": [
    {
      "Path": "srl_nokia-interfaces:interface[name=ethernet-1/31]",
      "values": {
        "srl_nokia-interfaces:interface": {
          "oper-state": "down"
        }
      }
    }
  ]
}
{
  "source": "clab-hackathon-spine1",
  "subscription-name": "default-1765311864",
  "timestamp": 1765312051571367211,
  "time": "2025-12-09T21:27:31.571367211+01:00",
  "updates": [
    {
      "Path": "srl_nokia-interfaces:interface[name=ethernet-1/1]",
      "values": {
        "srl_nokia-interfaces:interface": {
          "oper-state": "down"
        }
      }
    }
  ]
}
# Update is only sent on the impacted interfaces!
# on leaf1: set interface ethernet-1/31 admin-state enable
# commit
{
  "source": "clab-hackathon-spine1",
  "subscription-name": "default-1765311864",
  "timestamp": 1765312054359737797,
  "time": "2025-12-09T21:27:34.359737797+01:00",
  "updates": [
    {
      "Path": "srl_nokia-interfaces:interface[name=ethernet-1/1]",
      "values": {
        "srl_nokia-interfaces:interface": {
          "oper-state": "up"
        }
      }
    }
  ]
}
{
  "source": "clab-hackathon-leaf1",
  "subscription-name": "default-1765311864",
  "timestamp": 1765312054367128279,
  "time": "2025-12-09T21:27:34.367128279+01:00",
  "updates": [
    {
      "Path": "srl_nokia-interfaces:interface[name=ethernet-1/31]",
      "values": {
        "srl_nokia-interfaces:interface": {
          "oper-state": "up"
        }
      }
    }
  ]
}
# Streaming telemetry update is immediately sent, not on sampling interval - less than 5s have passed!
```

</details>

Why the difference in the behaviour between the two streaming telemetry sessions, though?

There is one knob we haven't touched yet, and that's `--stream-mode`. Since we didn't specify this, gNMIc let the device choose for us by default: this is the _TARGET_DEFINED_ mode.

The two modes for streaming telemetry with gNMI are: _SAMPLE_ and _ON_CHANGE_.  
In _SAMPLE_ mode, the streaming telemetry target sends an update (or _samples_) on the state of the path to the streaming telemetry collector. This is the sane default case for most counters.  
In _ON_CHANGE_ mode, the streaming telemetry target only sends an update for the target once it has _changed_. This is useful for paths which have a defined set of states (like up/down, enable/disable, protocol state machines like BGP, etc), and do not often change.

With the default _TARGET_DEFINED_ mode, we left the choice to the device. In our case, the output packet counters were sampled, while the interface oper-state was correctly reported only (and immediately!) on change.

Try manually setting the SAMPLE stream-mode on the previously tested interface operational state path, and see how wasteful it can be!

<details>
<summary>Task 50.2 part 4 solution</summary>

```json
$ gnmic -a clab-hackathon-leaf1,clab-hackathon-spine1 subscribe --stream-mode SAMPLE --path "/interface[name=ethernet-*]/oper-state"
```

</details>

## Task 50.3: gNMIc as a server

So far, we ran gNMIc as an interactive CLI tool. However, we can also start gNMIc as a server, exposing the data it collects to other services, in various formats! To do this, we will need to modify the configuration, and add some additional details:
- A list of [targets](https://gnmic.openconfig.net/user_guide/targets/targets/) to automatically subscribe to
- A list of [subscriptions](https://gnmic.openconfig.net/user_guide/subscriptions/) for paths, that can be assigned to targets
- An [output](https://gnmic.openconfig.net/user_guide/outputs/prometheus_output/) to expose

Let's create a configuration that connects to all SR Linux devices in our network, collects every 5 seconds in sample stream mode interface operational state, traffic-rate and statistics paths, and outputs to a Prometheus scrape endpoint, which should run on port 9273!

Use the [SR Linux YANG Browser](https://yang.srlinux.dev/v25.10.1) to help you find the correct YANG paths. 

<details>
<summary>Task 50.3 solution</summary>

```yaml
username: admin
password: NokiaSrl1!
port: 57400
timeout: 5s
skip-verify: true
encoding: json_ietf
targets:
  clab-hackathon-leaf1:
    subscriptions:
      - srl-if-stats
  clab-hackathon-leaf2:
    subscriptions:
      - srl-if-stats
  clab-hackathon-spine1:
    subscriptions:
      - srl-if-stats
  clab-hackathon-spine2:
    subscriptions:
      - srl-if-stats

subscriptions:
  srl-if-stats:
    mode: stream
    stream-mode: sample
    sample-interval: 5s
    paths:
      - /interface[name=ethernet-*]/oper-state
      - /interface[name=ethernet-*]/statistics
      - /interface[name=ethernet-*]/traffic-rate

outputs:
  prom-output:
    type: prometheus
    listen: :9273
```

Start the gNMIc server with `gnmic subscribe`
</details>

You should be able to hit the Prometheus scrape endpoint with `curl http://localhost:9273/metrics` and see your switches' metrics there! Don't be alarmed if the operational state is missing, though...

## Task 50.4: Processors

Try to look in the metrics output for the operational state! Even if you found the right path in the previous task, you won't be able to see it... Recall the manual commands we ran, scroll up in your terminal to see the format of the operational status (or check the solution): it's a string!

Prometheus does not have a "string" metric type, but rather requires all metrics to be some form of number (integer or floating-point number).

Thankfully, we can use _processors_ to pre-process our telemetry data before outputting it. The [string replace processor](https://gnmic.openconfig.net/user_guide/event_processors/event_strings/#replace) can do our work for us, replace "down" with "0" and "up" with "1".

To apply a processor on an output, it must be [referenced in it](https://gnmic.openconfig.net/user_guide/event_processors/intro/#linking-an-event-processor-to-an-output).

<details>
<summary>Task 50.4 solution</summary>

```yaml
# ...
processors:
  replace-operstate:
    event-strings:
      value-names:
        - ".*"
      transforms:
        - replace:
            apply-on: "value"
            old: "down"
            new: "0"
        - replace:
            apply-on: "value"
            old: "up"
            new: "1"

outputs:
  prom-output:
    type: prometheus
    listen: :9273
    event-processors:
      - replace-operstate
```

</details>

We can now see the oper-state metrics if we `curl` again:

```
$ curl http://localhost:9273/metrics 
# HELP srl_nokia_interfaces_interface_oper_state gNMIc generated metric
# TYPE srl_nokia_interfaces_interface_oper_state untyped
srl_nokia_interfaces_interface_oper_state{interface_name="ethernet-1/1",source="clab-hackathon-leaf1",subscription_name="srl-if-stats"} 1
srl_nokia_interfaces_interface_oper_state{interface_name="ethernet-1/1",source="clab-hackathon-leaf2",subscription_name="srl-if-stats"} 1
srl_nokia_interfaces_interface_oper_state{interface_name="ethernet-1/1",source="clab-hackathon-spine1",subscription_name="srl-if-stats"} 1
srl_nokia_interfaces_interface_oper_state{interface_name="ethernet-1/1",source="clab-hackathon-spine2",subscription_name="srl-if-stats"} 1
srl_nokia_interfaces_interface_oper_state{interface_name="ethernet-1/10",source="clab-hackathon-leaf1",subscription_name="srl-if-stats"} 0
srl_nokia_interfaces_interface_oper_state{interface_name="ethernet-1/10",source="clab-hackathon-leaf2",subscription_name="srl-if-stats"} 0
```

## Task 50.4: Putting gNMIc server into our topology

The gNMIc server should not be running separately from our Containerlab topology - after all, the nice thing about these topologies is that they are reproducible, let's leverage that!

Thankfully, there is a pre-made [gNMIc container image](https://github.com/openconfig/gnmic/pkgs/container/gnmic) that we can use. How do we pass the configuration file to gNMIc, though?  
We can use the `binds` node property to mount files into the container's file system, similarly to Docker's bind mount capability with `-v` - see more information on this feature [here](https://containerlab.dev/manual/nodes/#binds). Instead of mounting the configuration file from our home folder, copy it to `config/gnmic.yaml` first though.

Due to the way the gNMIc container image is put together, you will also have to specify a command (`cmd`) to run: in this case, this is appended as a suffix to the `gnmic` command inside the container - it's enough to pass only the configuration file as an option, and the subscribe gNMIc command.

<details>
<summary>Task 50.4 solution</summary>

```yaml
name: hackathon
topology:
  nodes:
#   ...
    gnmic:
      kind: linux
      image: ghcr.io/openconfig/gnmic:0.42.1
      binds:
        - config/gnmic.yaml:/.gnmic.yaml
      cmd: "--config /.gnmic.yaml subscribe"
#   ...
```
</details>

You might ask though - how will gNMIc know the IP addresses of the switches ahead of time? We don't need to know them, we can just refer to the container's hostname instead when querying from inside a container (within the same container network, of course!), which will automatically resolve to the container's address. Make sure to change this in the configuration file before you go ahead with deploying the new topology!

Verify that it works by redeploying and running `curl http://clab-hackathon-gnmic:9273/metrics` on the host!


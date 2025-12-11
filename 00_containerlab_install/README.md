## Intro to Containerlab

### Task 00.1: Prerequisites

To install Containerlab, you will need a Linux system that's recently new (running kernel 5.4 or newer).

If you would like to use Containerlab on a non-Linux host, for Windows, WSL2 is the recommended way of running it, while for Mac, any Linux VM solution (`lima`, OrbStack, UTM, etc) works well.

_For the purposes of this workshop, please use the VMs we have provided and hosted for you._

### Task 00.2: Installing Containerlab

Let's go through the quick install procedure!

The [quick-start script](https://containerlab.dev/quickstart/) on the Containerlab website works with most DEB (Debian, Ubuntu) and RPM-based (Red Hat, CentOS, Fedora, etc) distributions.

After logging in, run the following command:

```
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

This should install Containerlab for you!

> [!TIP]
> If you would like to use Containerlab without sudo, like the output of the installer suggests, you should add your user to this group using the following command:  
> 
> `gpasswd -a <your username here> clab_admins`
> 
> and apply the new group membership to your terminal session using the command
> `newgrp clab_admins` (or log in and out from the SSH session)

The Containerlab installation can be tested by running `containerlab version`:
```
$ containerlab version
  ____ ___  _   _ _____  _    ___ _   _ _____ ____  _       _
 / ___/ _ \| \ | |_   _|/ \  |_ _| \ | | ____|  _ \| | __ _| |__
| |  | | | |  \| | | | / _ \  | ||  \| |  _| | |_) | |/ _` | '_ \
| |__| |_| | |\  | | |/ ___ \ | || |\  | |___|  _ <| | (_| | |_) |
 \____\___/|_| \_| |_/_/   \_\___|_| \_|_____|_| \_\_|\__,_|_.__/

    version: 0.71.0
     commit: 7ef796f07
       date: 2025-10-10T17:36:13Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.71/
```

### Task 00.3: Testing Containerlab

In order to test and validate your Containerlab installation, you can deploy a "hello world" topology example and test if traffic passes between two nodes.

The following command clones the provided Git repository, and deploys the Containerlab topology contained within.

```
clab deploy -t https://github.com/vista-/clab-helloworld
```

Once deployed, you should be able to ping from `host1` to `host2`.

```
$ docker exec clab-helloworld-host1 ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=1.037 ms
```

Finally, we can clean up the topology we just created by doing

```
clab destroy -t https://github.com/vista-/clab-helloworld
```

**What did we learn from this?**
- Containerlab has a command alias, `clab`, which is quicker to type.
- Containerlab topologies can be deployed from remote URLs! GitHub repositories, direct URLs to Containerlab topologies, even S3 links work fine.
- `docker exec` can be used to run commands on containers deployed by Containerlab.  
This is one of the two main methods for interacting with a node in a running Containerlab topology.


### Task 00.4: Creating your first Containerlab topology

#### Terminology

The most important objects within a Containerlab topology definition are:

- Nodes  
Within the `nodes` section of a Containerlab topology definition the actual containers that should be deployed are described. In most cases, each node represents a single container. This could be a network element (router, switch, firewall, etc), but can also be any other Linux-based container.
- Links  
In order to create an actual topology, the defined nodes need to be interconnected. In the `links` section, we can define links by providing pairs of `endpoints` for Containerlab to connect. An endpoint consist of a `node` and an interface of the node.

References:
- [Containerlab nodes](https://containerlab.dev/manual/nodes/)
- [Containerlab links](https://containerlab.dev/manual/topo-def-file/#links)

#### The first lab topology

Containerlab supports [over 35 different types of network OSes](https://containerlab.dev/manual/kinds/) that can be ran inside a topology. However, most commercial NOSes can only be downloaded with an active account for a given vendor's website or download portal.

For the purposes of creating our first Containerlab topology, we will be recreating a similar "hello world" topology as we tested with before.

The topology, called "test", will consists of two Linux containers, `test1` and `test2`. In a Containerlab topology node Kinds are used to specify how Containerlab should manage and set up the node for use during deployment. In case of a Linux container, this kind is simply called [`linux`](https://containerlab.dev/manual/kinds/linux/).

We also need to specify what generic Linux container we are going to be running - the container image will determine this. 

A very small, and lightweight container image that's frequently used in the industry to build more complex applications on top is based on Alpine Linux. This container image is [hosted in Docker Hub](https://hub.docker.com/_/alpine), which means that you only need to specify the name of the container: `alpine`!

By default, Docker will always pull the "latest" version of the container image (tagged with "latest"), but in most cases, it's better to specify a specific container image version (or **tag**) that we want to use in a topology. This way, changes between versions won't impact your workflow, even if you deploy the same topology years later.

Let's use the version 3.21 of Alpine Linux in the topology!

The two nodes in the topology should be interconnected by a single link - let's use the first _data-plane interface_ on each container to make the connection.

With Containerlab, all nodes (by default) come with a management connection. Some network OSes specify a separate interface for management connectivity, but for those that do not, the rule of thumb is that Containerlab will use the first interface for that purpose. As an example, check out the [VyOS kind documentation](https://containerlab.dev/manual/kinds/vyosnetworks_vyos/).

In case of Linux containers, that first interface that gets reserved for management connectivity (and therefore cannot be used in the topology wiring) is `eth0`. Every other interface is free to use.

Your task is to create this Containerlab topology by writing your own YAML topology definition (it should be a file ending with `.clab.yaml`), deploy it, set the IP addresses 192.168.1.1/24 and 192.168.1.2/24 on the respective Linux container, and perform a ping!

Don't forget to `clab destroy` the test topology once you are done working with it. 

<details>
<summary>Task 00.4 solution</summary>

Containerlab Topology:
```yaml
name: test

topology:
  nodes:
    test1:
      kind: linux
      image: alpine:3.21
    test2:
      kind: linux
      image: alpine:3.21

  links:
    - endpoints: ["test1:eth1", "test2:eth1"]
```

Deployment:
```
clab deploy -t test.clab.yaml
```

IP address assignment and ping:
```shell
# Enter the container shell by running a docker exec command:
workshop-123$ docker exec -it clab-test-test1 sh

/$ ip addr add 192.168.1.1/24 dev eth1
# do this on test2 as well
/$ ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: seq=0 ttl=64 time=1.318 ms
```

Cleanup:
```
clab destroy -t test.clab.yaml
```

</details>


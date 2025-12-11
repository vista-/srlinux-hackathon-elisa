# Saving your work in Containerlab

## Task 12.1: Diffs, applying configs and checkpoints

Inside SR Linux, you can `diff` between different versions of your configuration.

In _candidate_ mode, the _running_ configuration is used as a baseline for comparison.

```
--{ candidate shared default }--[  ]--
A:admin@leaf1# set interface ethernet-1/1 description "test"

--{ * candidate shared default }--[  ]--
A:admin@leaf1# diff
      interface ethernet-1/1 {
+         description test
      }
```

You can also request the diff to be shown in a `flat` format - this shows you the difference in the form of `set`, `insert` or `delete` commands that you can paste into the CLI.

```
--{ * candidate shared default }--[  ]--
A:admin@leaf1# diff flat
insert / interface ethernet-1/1 description test
```

The `flat` keyword can also be used with the `info` command, which shows you the configuration in the current context.

A key feature in the configuration management that will help us is called _checkpoints_.

Checkpoints can be created in SR Linux at commits by including the `checkpoint` keyword to the `commit` command.

Now, you might think - this would have been nice to know earlier, I didn't use the `checkpoint` keyword to commit my changes!  
Thankfully, we are saved by Containerlab in this situation, since Containerlab creates a checkpoint after it deploys the node, called `clab-initial`.

Checkpoints are kept track of using name and relative IDs. The most recent checkpoint is always the ID 0, and each older checkpoint has incrementing IDs - for example, if you want to diff between the current and previous checkpoint, you would `diff` against checkpoint ID 1.

**Using this knowledge, try to capture the changes you have made so far!**

## Task 12.2: Automatic checkpointing

The SR Linux configuration management is very flexible, and its behaviour can be configured. For example, there is a setting to automatically generate checkpoints on each commit.  
This is documented in the SR Linux documentation.

**Configure this setting, and use the newly generated checkpoint to diff against the previous checkpoint!**

<details>
<summary>Task 12.2 solution</summary>

```
--{ * candidate shared default }--[  ]--
A:admin@leaf1# set system configuration auto-checkpoint true

--{ * candidate shared default }--[  ]--
A:admin@leaf1# commit stay
/system:
    Generated checkpoint '/etc/opt/srlinux/checkpoint/checkpoint-0.json' with name 'checkpoint-2025-12-04T14:15:38.172Z' and comment 'automatic checkpoint after commit 3'

All changes have been committed. Starting new transaction.

--{ + candidate shared default }--[  ]--
A:admin@leaf1# diff flat checkpoint 1
insert / system configuration auto-checkpoint true
insert / interface ethernet-1/1 description test
```
</details>

## Task 12.3: SR Linux configuration management

In SR Linux, committing a change does not save your work automatically.  
Instead, the changed running configuration will only be persisted when you issue the `save startup` command.

To help you keep track of the state of the configuration, there is a helpful hint in the prompt in the forms of the *asterisk* and *plus* sign.

The *asterisk* marks an uncommitted change. Generally, you will only see this in `candidate` mode.  
The *plus* marks an unsaved change in the running configuration.

Here is an example session to help showcase this:

```
--{ candidate shared default }--[  ]--
A:admin@leaf1# set interface ethernet-1/1 description "test"

# We changed the candidate config, but haven't committed it yet
--{ * candidate shared default }--[  ]--
A:admin@leaf1# commit stay
All changes have been committed. Starting new transaction.

# We have unsaved changes in the running configuration
--{ + candidate shared default }--[  ]--
A:admin@leaf1# set interface ethernet-1/1 description "hello"

# We have both unsaved changes in the running config and uncommitted changes in the candidate config
--{ +* candidate shared default }--[  ]--
A:admin@leaf1# commit now

All changes have been committed. Leaving candidate mode.

# Only unsaved changes in the running config now
--{ + running }--[  ]--
A:admin@leaf1# save startup

/system:
    Saved current running configuration as initial (startup) configuration '/etc/opt/srlinux/config.json'

# All good!
--{ running }--[  ]--
A:admin@leaf1# 
```

However, much like the automatic checkpoints, there is a knob in the file system to let you automatically save your work.

Your configuration is persisted to disk under the `/etc/opt/srlinux/config.json` path. This is a JSON file, containing not only your configuration, but additional information about SR Linux version, loaded YANG models, etc.

## Task 12.4: Adding startup configs to your lab topology

When working with network labs, you might not want to start from scratch every time you deploy a lab. Even if you have the configurations saved, you probably don't want to apply them by hand node by node...
    
Instead of supplying configuration by hand, we can also let Containerlab load the startup configurations for us!
    
Depending on the `kind` of the node, there might be several different ways on how to add a startup config to the simulated network element. These are all documented under the [Kinds section](https://containerlab.dev/manual/kinds/) of the Containerlab documentation.
    
>[!WARNING]
> As a reminder: Containerlab applies some initial default configurations to some NOS's already. This is important to know when working with startup-configs. 
> 
> A general rule of thumb is leave out management plane configuration, so management connectivity, SSH, authentication, etc, to avoid situations where Containerlab and your own startup configuration might want to modify the same configuration, and leave you with a node that you cannot connect to.
    
In SR Linux, there are two ways of adding a startup config to a node:
- as a partial config added on top of the default Containerlab-added management configuration, either in the form of CLI commands, or in a hierarchical configuration format

- or as a full SR Linux configuration file in, JSON format (note: the hierarchical config format is not a JSON!). This is the format you can find on SR Linux nodes' file systems

More information about SR Linux startup configs can be found in the [SR Linux Containerlab documentation](https://containerlab.dev/manual/kinds/srl/).

In addition to file mounts, it is also possible to automatically execute CLI commands after container startup. This is useful for Linux client devices, to set up Linux interfaces' IP addressing, or create a bond interface, for example.

```yaml
name: linux-cmd-exec
topology:
  nodes:
    test1:
      kind: linux
      image: alpine
      exec:
        - ip link set dev eth1 up
        - ip addr add 192.168.1.1/24 dev eth1
```

**Your next task is to re-deploy the topology with the configuration used in the previous step now in the startup configuration files!**  
The files should be stored in the `./config` directory, and a separate startup configuration file should be created for each SR Linux node.

> [!IMPORTANT]
> Before you deploy the topology again, you should **save your work**!

Let's take a quick look our working directory first, before we destroy the topology!

You can notice that a new directory prefixed with `clab-` has been created. This is called the _lab directory_, and is created on the first deployment of a topology by Containerlab. For node kinds that support it (such as SR Linux), this is where the state of the network OS is persisted between deployments.
    
For example, take a look at the `clab-workshop/leaf1/config/config.json` file:

```json
{
  "_preamble": {
    "header": {
      "generated-by": "SRLINUX",
      "name": "",
      "comment": "",
      "created": "2025-12-05T12:21:01.080Z",
      "release": "v25.10.1",
  ...
}
```

This is exactly the file that you would see on the SR Linux node `leaf1` under the `/etc/opt/srlinux/config.json` path that was mentioned earlier! This is because part of the SR Linux node's file system is mounted into the container from the lab directory - and this is how nodes persist their configuration between deployments.

Many other NOSes also support saving the node configuration to the lab directory with the `containerlab save` command.

However, packaging the entire lab directory is undesired, as it can contain much more state than normally needed to recover the NOSes state - normally, all you need is a startup-config. 
    
Using the `-c` command-line flag in `clab deploy` and `clab destroy` commands removes the lab directory created by Containerlab on lab deployment, so let's use the `containerlab destroy -c` command to destroy the topology and remove the lab directory!
    
Once you are done with adding the startup configuration files to the topology, it's time to redeploy again! Pinging should still work :)
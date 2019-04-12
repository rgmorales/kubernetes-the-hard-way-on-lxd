# Prerequisites

## Initializing LXD

If you never used LXD on your host, you need to initialize it:

```
lxd init
```

Create a new storage pool, and selecr the backend to be dir, this is the only supported backend for our tutorial.
During the creation of the network bridge, do not select ipv6, typing none for it. Your command should look like this:

```
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]: lxd-storage
Name of the storage backend to use (btrfs, dir, lvm) [default=btrfs]: dir
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: none
Would you like LXD to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```

You can now check the lxc containers by running:

```
lxc list
```

You should see no containers created at this point.

## Creating containers profiles

We will use a special profile to run our containers, since some components require special access to modules to run. This is not safe for a production environment, and should be used only for this lab.
More info [here](https://github.com/juju-solutions/bundle-canonical-kubernetes/wiki/Deploying-on-LXD).

create the profile configuration yaml with the following content:

```
cat <<EOF |tee kube-profile.yaml 
config:
  limits.cpu: "1"
  limits.memory.swap: "false"
  boot.autostart: "false"
  linux.kernel_modules: ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter
  raw.lxc: |
    lxc.apparmor.profile=unconfined
    lxc.mount.auto=proc:rw sys:rw cgroup:rw
    lxc.cgroup.devices.allow=a
    lxc.cap.drop=
  security.nesting: "true"
  security.privileged: "true"
description: ""
devices:
  aadisable:
    path: /sys/module/nf_conntrack/parameters/hashsize
    source: /dev/null
    type: disk
  aadisable1:
    path: /sys/module/apparmor/parameters/enabled
    source: /dev/null
    type: disk
EOF
```

Now create the profile:

```
 lxc profile create kube-profile
```

Set the profile with the properties from the yaml file:

```
cat kube-profile.yaml | lxc profile edit kube-profile
```

Check the profile content with:

```
lxc profile show kube-profile
``` 

Next: [Installing the Client Tools](02-client-tools.md)

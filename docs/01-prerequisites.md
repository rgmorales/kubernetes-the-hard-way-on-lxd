# Prerequisites

## Initializing LXD

If you never used LXD on your host, you need to initialize it:
Ensure you have lxc version 3.0 and above; Note: These lxc instructions does not work in version 2.x

### Note:

By default, if you have used multipass, lxd and lxc is already installed.

```
lxc --version
3.0.3

```

You may ignore if you already have lxc installed. You can skip the next step and directly go to creating storage pool
Just a little notes on installing the right version on ubuntu 18.04

```
sudo apt clean
sudo apt install -t xenial-backports lxd lxd-client
sudo apt update
```

Create a new storage pool, and select the backend to be dir, this is the only supported backend for this tutorial.

```
lxc storage create lxd-storage dir

```

You can now check the lxd storage by running:

```
ubuntu@k8s-hardway-18:~$ lxc storage list
+-------------+-------------+--------+----------------------------------------+---------+
|    NAME     | DESCRIPTION | DRIVER |                 SOURCE                 | USED BY |
+-------------+-------------+--------+----------------------------------------+---------+
| lxd-storage |             | dir    | /var/lib/lxd/storage-pools/lxd-storage | 0       |
+-------------+-------------+--------+----------------------------------------+---------+
```

You should see no containers created at this point.

## Creating containers profiles

We will use a special profile to run our containers, since some components require special access to modules to run. This is not safe for a production environment, and should be used only for this lab.
More info [here](https://github.com/juju-solutions/bundle-canonical-kubernetes/wiki/Deploying-on-LXD).

create the profile configuration yaml with the following content:

```
cat <<EOF |tee kube-profile.yaml
config:
  limits.cpu: "2"
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

Disable swap on your host:

```
sudo swapoff -a
```

Next: [Installing the Client Tools](02-client-tools.md)

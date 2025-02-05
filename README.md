# Kubernetes The Hard Way on LXD

This tutorial is based on Kelsey's Hightower tutorial to deploy Kubernetes the hard way, but using LXC containers in a single virtual host.
Some modifications in the config files are required in order to run all servers in a single node.
One major reason for bottleneck can be the disk IO needed, making a SSD or M.2 card a must. While deploying etcd on 3 containers, you will see a lot of io requests to store and retrieve data. Spinning disks will make this lab impossible to deploy.

The original excellent guide from Kelsey can be found here: [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way). This guide is an adaptation to his guide, and many steps are exactly the same.
This guide has some shell scripts to execute operations on containers. While executing those commands, take in consideration what would you do in several production servers.

# Kubernetes The Hard Way

This tutorial walks you through setting up Kubernetes the hard way in a single host using LXC containers. This guide is not for people looking for a fully automated command to bring up a Kubernetes cluster. If that's what you are looking for, then check out [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine), or the [Getting Started Guides](http://kubernetes.io/docs/getting-started-guides/). For a complete deployment using juju on LXD, you can use [conjure](https://tutorials.ubuntu.com/tutorial/install-kubernetes-with-conjure-up#0) Please note that the deployment with conjure is slightly differen than this tutorial and uses different components and versions.

Kubernetes The Hard Way is optimized for learning, which means taking the long route to ensure you understand each task required to bootstrap a Kubernetes cluster.

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that stop you from learning!

You can try this tutorial in a VM created with Virtualbox, make sure to create the VM in a host with and SSD or M.2 card for storage.

or you can try this tutorial using multipass (https://multipass.run/)

## Target Audience

The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together.

## Cluster Details

Kubernetes The Hard Way guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption between components and RBAC authentication.

- [Kubernetes](https://github.com/kubernetes/kubernetes) 1.22.3
- [containerd Container Runtime](https://github.com/containerd/containerd) 1.4.4
- [gVisor](https://github.com/google/gvisor) 50c283b9f56bb7200938d9e207355f05f79f0d17
- [CNI Container Networking](https://github.com/containernetworking/cni) 0.9.1
- [etcd](https://github.com/coreos/etcd) v3.4.15
- [CoreDNS](https://github.com/coredns/coredns) v1.8

## Labs

This tutorial assumes you have a server with Ubuntu 20.04, and an SSD or M.2 disk where the containers will be running.

- [Multipass Instructions](docs/00-private-cloud-prerequisites.md)
- [Prerequisites](docs/01-prerequisites.md)
- [Installing the Client Tools](docs/02-client-tools.md)
- [Compute Resources](docs/03-compute-resources.md)
- [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
- [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
- [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
- [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
- [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
- [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
- [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
- [Deploying the DNS Cluster Add-on](docs/11-dns-addon.md)
- [Smoke Test](docs/12-smoke-test.md)
- [Cleaning Up](docs/13-cleanup.md)

I ran using Multipass; If you have very little memory (say 8GB), I recommend this option.
[https://multipass.run](https://multipass.run/)
I was able to run the entire k8s cluster (3+1+3 nodes) with less than 2GB.

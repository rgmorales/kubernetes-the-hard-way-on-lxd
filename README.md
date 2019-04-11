# Kubernetes The Hard Way on LXD
This tutorial is based on Kelsey's tutorial to deploy Kubernetes the hard way, but using LXC containers in a single host. 
Some modifications in the config files are required in order to run all servers in a single node.
One major reason for bottleneck can be the disk IO needed, making an SSD or M.2 card a must. While deploying etcd on 3 containers, you will see a lot of io requests to store and retrieve data. Spinning disks will make this impossible to deploy.

The original excellent guide from Kelsey can be found here: [Kubernetes the Hard Way] (https://github.com/kelseyhightower/kubernetes-the-hard-way)

You can also deploy Kubernetes using juju and conjure, but that is the easy way ;)

# Kubernetes The Hard Way

This tutorial walks you through setting up Kubernetes the hard way in a single host using LXC containers. This guide is not for people looking for a fully automated command to bring up a Kubernetes cluster. If that's what you are looking for, then check out [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine), or the [Getting Started Guides](http://kubernetes.io/docs/getting-started-guides/). For a complete deployment using juju on LXD, you can use [conjure](https://tutorials.ubuntu.com/tutorial/install-kubernetes-with-conjure-up#0) Please note that the deployment with conjure is slightly differen than this tutorial and uses different components and versions.

Kubernetes The Hard Way is optimized for learning, which means taking the long route to ensure you understand each task required to bootstrap a Kubernetes cluster.

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that stop you from learning!

## Target Audience

The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together.

## Cluster Details

Kubernetes The Hard Way guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption between components and RBAC authentication.

* [Kubernetes](https://github.com/kubernetes/kubernetes) 1.12.0
* [containerd Container Runtime](https://github.com/containerd/containerd) 1.2.0-rc.0
* [gVisor](https://github.com/google/gvisor) 50c283b9f56bb7200938d9e207355f05f79f0d17
* [CNI Container Networking](https://github.com/containernetworking/cni) 0.6.0
* [etcd](https://github.com/coreos/etcd) v3.3.9
* [CoreDNS](https://github.com/coredns/coredns) v1.2.2

## Labs

This tutorial assumes you have a server with Ubuntu 18.04 installed, and an SSD or M.2 disk where the containers will be running.

* [Prerequisites](docs/01-prerequisites.md)
* [Installing the Client Tools](docs/02-client-tools.md)
* [Creating Containers](docs/03-creating-containers.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)

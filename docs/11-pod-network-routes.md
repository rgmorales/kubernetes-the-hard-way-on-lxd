# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address for the workers, by checking the networking on the lxc containers:
```
lxc list 
```

> output

```
+--------------+---------+-------------------+------+------------+-----------+
|     NAME     |  STATE  |       IPV4        | IPV6 |    TYPE    | SNAPSHOTS |
+--------------+---------+-------------------+------+------------+-----------+
| controller-0 | RUNNING | 10.0.2.10 (eth1)  |      | PERSISTENT | 0         |
|              |         | 10.0.1.39 (eth0)  |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+
| controller-1 | RUNNING | 10.0.2.11 (eth1)  |      | PERSISTENT | 0         |
|              |         | 10.0.1.252 (eth0) |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+
| controller-2 | RUNNING | 10.0.2.12 (eth1)  |      | PERSISTENT | 0         |
|              |         | 10.0.1.111 (eth0) |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+
| haproxy      | RUNNING | 10.0.1.100 (eth0) |      | PERSISTENT | 0         |
+--------------+---------+-------------------+------+------------+-----------+
| worker-0     | RUNNING | 10.1.0.1 (cnio0)  |      | PERSISTENT | 0         |
|              |         | 10.0.2.20 (eth1)  |      |            |           |
|              |         | 10.0.1.37 (eth0)  |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+
| worker-1     | RUNNING | 10.1.1.1 (cnio0)  |      | PERSISTENT | 0         |
|              |         | 10.0.2.21 (eth1)  |      |            |           |
|              |         | 10.0.1.134 (eth0) |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+
| worker-2     | RUNNING | 10.1.2.1 (cnio0)  |      | PERSISTENT | 0         |
|              |         | 10.0.2.22 (eth1)  |      |            |           |
|              |         | 10.0.1.187 (eth0) |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+
```
The internal IPs are on the eth1 network adapter.

## Routes

Create network routes for each worker instance:

```
for i in 0 1 2; do
  lxc exec worker-${i} -- ip route add 10.200.0.0/24 via 10.0.2.20 dev eth1
done
```

List the routes in all workers:

```
for i in 0 1 2; do
  lxc exec worker-${i} -- ip route 
done
```

> output

```
NAME                            NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-081879136902de56  kubernetes-the-hard-way  10.240.0.0/24  kubernetes-the-hard-way   1000
default-route-55199a5aa126d7aa  kubernetes-the-hard-way  0.0.0.0/0      default-internet-gateway  1000
kubernetes-route-10-200-0-0-24  kubernetes-the-hard-way  10.200.0.0/24  10.240.0.20               1000
kubernetes-route-10-200-1-0-24  kubernetes-the-hard-way  10.200.1.0/24  10.240.0.21               1000
kubernetes-route-10-200-2-0-24  kubernetes-the-hard-way  10.200.2.0/24  10.240.0.22               1000
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)

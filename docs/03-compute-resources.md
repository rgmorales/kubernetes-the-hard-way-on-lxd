# Creating the LXC containers

##Creating bridges

Create the two interfaces to be used by the containers, one interface has NAT and the other is going to be our internal network:

External interface:
```
lxc network create kube0 ipv6.address=none ipv4.address=10.0.1.1/24 ipv4.nat=true
```

Internal interface:
```
lxc network create kube1 ipv6.address=none ipv4.address=10.0.2.1/24 ipv4.nat=false
```

We will now create the lxc containers 

## Controllers

Create the three controllers:
```
for i in 0 1 2; do
  lxc launch images:ubuntu/18.04/amd64 controller-${i} -p kube-profile -s lxd-storage
done
```

## Workers

Create the 3 workers:
```
for i in 0 1 2; do
  lxc launch images:ubuntu/18.04/amd64 worker-${i} -p kube-profile -s lxd-storage
done

```

Check if all the containers are created:
```
lxc list
```

All containers should be running, but they have no network assigned to them. You should have a message during container creation stating that they were created with no network:

```
+--------------+---------+------+------+------------+-----------+
|     NAME     |  STATE  | IPV4 | IPV6 |    TYPE    | SNAPSHOTS |
+--------------+---------+------+------+------------+-----------+
| controller-0 | RUNNING |      |      | PERSISTENT | 0         |
+--------------+---------+------+------+------------+-----------+
| controller-1 | RUNNING |      |      | PERSISTENT | 0         |
+--------------+---------+------+------+------------+-----------+
| controller-2 | RUNNING |      |      | PERSISTENT | 0         |
+--------------+---------+------+------+------------+-----------+
| worker-0     | RUNNING |      |      | PERSISTENT | 0         |
+--------------+---------+------+------+------------+-----------+
| worker-1     | RUNNING |      |      | PERSISTENT | 0         |
+--------------+---------+------+------+------------+-----------+
| worker-2     | RUNNING |      |      | PERSISTENT | 0         |
+--------------+---------+------+------+------------+-----------+
````

Stop all the containers:
```
lxc stop --all 
```

Attach the networks to the containers:
```
for i in 0 1 2; do
  lxc network attach kube1 controller-${i}
  lxc network attach kube0 controller-${i}
  lxc network attach kube1 worker-${i}
  lxc network attach kube0 worker-${i}
done
```

Now we will create the yaml files for networking for each container, and push the file to the container. After that, we will apply the networking configuration on each container:
```
for i in 0 1 2; do

cat <<EOF |tee 10-lxc.yaml 
network:
  version: 2
  ethernets:
    eth0:
       dhcp4: no
       addresses: [10.0.1.1${i}/24]
       gateway4: 10.0.1.1
       nameservers:
         addresses: [8.8.8.8,8.8.4.4]
    eth1:
       dhcp4: no
       addresses: [10.0.2.1${i}/24]       
EOF

sudo lxc file push 10-lxc.yaml controller-${i}/etc/netplan/

lxc exec controller-${i} -- sudo netplan apply

cat <<EOF |tee 10-lxc.yaml 
network:
  version: 2
  ethernets:
    eth0:
       dhcp4: no
       addresses: [10.0.1.2${i}/24]
       gateway4: 10.0.1.1
       nameservers:
         addresses: [8.8.8.8,8.8.4.4]
    eth1:
       dhcp4: no
       addresses: [10.0.2.2${i}/24]       
EOF

sudo lxc file push 10-lxc.yaml worker-${i}/etc/netplan/

lxc exec worker-${i} -- sudo netplan apply

done
```

Now, stop all containers and start all of them:
```
lxc stop --all
```

```
lxc start --all
```

Now list all the containers and check for the network configurations:
```
 lxc list
+--------------+---------+------------------+------+------------+-----------+
|     NAME     |  STATE  |       IPV4       | IPV6 |    TYPE    | SNAPSHOTS |
+--------------+---------+------------------+------+------------+-----------+
| controller-0 | RUNNING | 10.0.2.10 (eth1) |      | PERSISTENT | 0         |
|              |         | 10.0.1.10 (eth0) |      |            |           |
+--------------+---------+------------------+------+------------+-----------+
| controller-1 | RUNNING | 10.0.2.11 (eth1) |      | PERSISTENT | 0         |
|              |         | 10.0.1.11 (eth0) |      |            |           |
+--------------+---------+------------------+------+------------+-----------+
| controller-2 | RUNNING | 10.0.2.12 (eth1) |      | PERSISTENT | 0         |
|              |         | 10.0.1.12 (eth0) |      |            |           |
+--------------+---------+------------------+------+------------+-----------+
| worker-0     | RUNNING | 10.0.2.20 (eth1) |      | PERSISTENT | 0         |
|              |         | 10.0.1.20 (eth0) |      |            |           |
+--------------+---------+------------------+------+------------+-----------+
| worker-1     | RUNNING | 10.0.2.21 (eth1) |      | PERSISTENT | 0         |
|              |         | 10.0.1.21 (eth0) |      |            |           |
+--------------+---------+------------------+------+------------+-----------+
| worker-2     | RUNNING | 10.0.2.22 (eth1) |      | PERSISTENT | 0         |
|              |         | 10.0.1.22 (eth0) |      |            |           |
+--------------+---------+------------------+------+------------+-----------+
```

You can check if the containers can ping each other:
```
lxc exec worker-0 -- ping 10.0.2.22
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

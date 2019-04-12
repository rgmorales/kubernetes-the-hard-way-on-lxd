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




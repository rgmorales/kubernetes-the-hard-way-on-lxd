# Deploying the DNS Cluster Add-on

In this lab you will deploy the [DNS add-on](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) which provides DNS based service discovery, backed by [CoreDNS](https://coredns.io/), to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on

Deploy the `coredns` cluster add-on:

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```

> output

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

## Note:

The replicaset needs to be bumped up from 2 to 3, with the above command there will be only 2 core-dns pods running.
Modify the core-dns deployment accordingly.

List all the containers, now you should see that the workers have an extra cni network configured:

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
| worker-2     | RUNNING | 10.0.2.22 (eth1)  |      | PERSISTENT | 0         |
|              |         | 10.0.1.187 (eth0) |      |            |           |
+--------------+---------+-------------------+------+------------+-----------+

```

List the pods created by the `kube-dns` deployment:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system

```

> output

```

NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s

```

## Verification

Create a `busybox` deployment:

```

kubectl run busybox --image=busybox:1.28 --command -- sleep 3600

```

List the pod created by the `busybox` deployment:

```

kubectl get pods -l run=busybox

```

> output

```

NAME                      READY   STATUS    RESTARTS   AGE
busybox-bd8fb7cbd-vflm9   1/1     Running   0          10s

```

Retrieve the full name of the `busybox` pod:

```

POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")

```

Execute a DNS lookup for the `kubernetes` service inside the `busybox` pod:

```

kubectl exec -ti $POD_NAME -- nslookup kubernetes

```

> output

```

Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local

```

If you dont get expected ouput, try checking core-dns pods. There should be 3 instances running.

Next: [Smoke Test](12-smoke-test.md)

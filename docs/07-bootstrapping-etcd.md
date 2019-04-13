# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/coreos/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`. 
We will download all files in the main server, push all files to the controllers, and run commands on each controller.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [coreos/etcd](https://github.com/coreos/etcd) GitHub project:

```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` command line utility:

```
{
for instance in controller-0 controller-1 controller-2; do
  lxc file push etcd-v3.3.9-linux-amd64.tar.gz ${instance}/home/ubuntu/
  lxc exec ${instance} -- tar -xvf /home/ubuntu/etcd-v3.3.9-linux-amd64.tar.gz -C /home/ubuntu/ 
  lxc exec ${instance} -- mv /home/ubuntu/etcd-v3.3.9-linux-amd64/etcd /usr/local/bin/
  lxc exec ${instance} -- mv /home/ubuntu/etcd-v3.3.9-linux-amd64/etcdctl /usr/local/bin/
done  
}
```

### Configure the etcd Server

```
{
for instance in controller-0 controller-1 controller-2; do  
  lxc exec ${instance} -- mkdir -p /etc/etcd /var/lib/etcd
  lxc exec ${instance} -- cp /home/ubuntu/ca.pem /etc/etcd/ 
  lxc exec ${instance} -- cp /home/ubuntu/kubernetes-key.pem /etc/etcd/ 
  lxc exec ${instance} -- cp /home/ubuntu/kubernetes.pem /etc/etcd/   
done  
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance

Create the `etcd.service` systemd unit file for each of the controllers:

```
{
for instance in 0 1 2; do  
  INTERNAL_IP=10.0.2.1${instance}
  
  ETCD_NAME=controller-${instance}
  
  cat <<EOF | tee etcd.service
    [Unit]
    Description=etcd
    Documentation=https://github.com/coreos

    [Service]
    ExecStart=/usr/local/bin/etcd \\
      --name ${ETCD_NAME} \\
      --cert-file=/etc/etcd/kubernetes.pem \\
      --key-file=/etc/etcd/kubernetes-key.pem \\
      --peer-cert-file=/etc/etcd/kubernetes.pem \\
      --peer-key-file=/etc/etcd/kubernetes-key.pem \\
      --trusted-ca-file=/etc/etcd/ca.pem \\
      --peer-trusted-ca-file=/etc/etcd/ca.pem \\
      --peer-client-cert-auth \\
      --client-cert-auth \\
      --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-peer-urls https://${INTERNAL_IP}:2380 \\
      --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
      --advertise-client-urls https://${INTERNAL_IP}:2379 \\
      --initial-cluster-token etcd-cluster-0 \\
      --initial-cluster controller-0=https://10.0.2.10:2380,controller-1=https://10.0.2.11:2380,controller-2=https://10.0.2.12:2380 \\
      --initial-cluster-state new \\
      --data-dir=/var/lib/etcd
    Restart=on-failure
    RestartSec=5
    [Install]
    WantedBy=multi-user.target
  EOF

  lxc file push etcd.service ${ETCD_NAME}/etc/systemd/system/
  
done
}
```

### Start the etcd Server

```
{
for instance in controller-0 controller-1 controller-2; do  
  lxc exec ${instance} -- systemctl daemon-reload
  lxc exec ${instance} -- enable etcd
  lxc exec ${instance} -- start etcd  
}
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> output

```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)

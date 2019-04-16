# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

We will execute the commands in this lab  on each controller instance: `controller-0`, `controller-1`, and `controller-2`. Using the lxc commands for that, example:

```
lxc exec ${instance} -- ${command}
```

Create the Kubernetes configuration directory:

```
{
for instance in controller-0 controller-1 controller-2; do
  lxc exec ${instance} -- mkdir -p /etc/kubernetes/config
done
}
```

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  
  for instance in controller-0 controller-1 controller-2; do
    lxc file push kube-apiserver ${instance}/usr/local/bin/
    lxc file push kube-controller-manager ${instance}/usr/local/bin/
    lxc file push kube-scheduler ${instance}/usr/local/bin/
    lxc file push kubectl ${instance}/usr/local/bin/  
  done  
}
```

### Configure the Kubernetes API Server

```
{
for instance in controller-0 controller-1 controller-2; do
  lxc exec ${instance} -- mkdir -p /var/lib/kubernetes/  
  lxc file push ca.pem ${instance}/var/lib/kubernetes/
  lxc file push ca-key.pem ${instance}/var/lib/kubernetes/
  lxc file push kubernetes-key.pem ${instance}/var/lib/kubernetes/
  lxc file push kubernetes.pem ${instance}/var/lib/kubernetes/
  lxc file push service-account-key.pem ${instance}/var/lib/kubernetes/
  lxc file push service-account.pem ${instance}/var/lib/kubernetes/
  lxc file push encryption-config.yaml ${instance}/var/lib/kubernetes/  
 done  
}
```

The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

```
{
for instance in 0 1 2; do

INTERNAL_IP=10.0.2.1${instance}
  
cat <<EOF | tee kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.0.2.10:2379,https://10.0.2.11:2379,https://10.0.2.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

lxc file push kube-apiserver.service controller-${instance}/etc/systemd/system/

done
}
```


### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place and create the `kube-controller-manager.service` systemd unit file::

```
cat <<EOF | tee kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.0.2.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF


for instance in controller-0 controller-1 controller-2; do

  lxc file push kube-controller-manager.kubeconfig ${instance}/var/lib/kubernetes/
  lxc file push kube-controller-manager.service ${instance}/etc/systemd/system/
  
done
```




### Configure the Kubernetes Scheduler


Create the `kube-scheduler.yaml` and the `kube-scheduler.service` configuration files:

```
cat <<EOF | tee kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

cat <<EOF | tee kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Move all config files into place:
```
for instance in controller-0 controller-1 controller-2; do
  lxc file push kube-scheduler.kubeconfig ${instance}/var/lib/kubernetes/
  lxc file push kube-scheduler.service ${instance}/etc/systemd/system/
  lxc file push kube-scheduler.yaml ${instance}/etc/kubernetes/config/  
done
```


### Start the Controller Services

```
{
for instance in controller-0 controller-1 controller-2; do  
  lxc exec ${instance} -- systemctl daemon-reload
  lxc exec ${instance} -- systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  lxc exec ${instance} -- systemctl start kube-apiserver kube-controller-manager kube-scheduler
done
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

### Verification

In one of the controllers, run the following command:

```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

You will have to move the ```/home/ubuntu/``` folder to run this command.

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

Test the haproxy:

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.


Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

For this part of the lab, we will send the commands to haproxy, which will loadbalance to the controllers, for that, update the admin.kubeconfig to the haproxy address on the server you used to create the lxc containers, do not change the file on any of the controllers:

```
 vi admin.kubeconfig
```

Change the field server to 10.0.1.100:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data:<CERTIFICATE DATA NOT REPRODUCED HERE>
    server: https://10.0.1.100:6443
  name: kubernetes-the-hard-way
contexts:
- context:
    cluster: kubernetes-the-hard-way
    user: admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data:<CERTIFICATE DATA NOT REPRODUCED HERE>
    client-key-data:<CERTIFICATE DATA NOT REPRODUCED HERE>
```  
  
Now you can create the roles on the cluster:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the `kubernetes` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

### Verification

Make a HTTP request for the Kubernetes version info on the haproxy:

```
curl --cacert ca.pem https://10.0.1.100:6443/version
```

> output

```
{
  "major": "1",
  "minor": "12",
  "gitVersion": "v1.12.0",
  "gitCommit": "0ed33881dc4355495f623c6f22e7dd0b7632b7c0",
  "gitTreeState": "clean",
  "buildDate": "2018-09-27T16:55:41Z",
  "goVersion": "go1.10.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)

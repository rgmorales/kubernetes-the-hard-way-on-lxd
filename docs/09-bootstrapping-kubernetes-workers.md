# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [gVisor](https://github.com/google/gvisor), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

Install the OS dependencies:

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- apt-get update
  lxc exec ${instance} -- apt-get -y install socat conntrack ipset
done
}
```

> The socat binary enables support for the `kubectl port-forward` command.

### Download and Install Worker Binaries

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.22.3/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.22.3/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.22.3/bin/linux/amd64/kubelet
```

Create the installation directories:

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- mkdir -p /etc/cni/net.d
  lxc exec ${instance} -- mkdir -p /opt/cni/bin
  lxc exec ${instance} -- mkdir -p /var/lib/kubelet
  lxc exec ${instance} -- mkdir -p /var/lib/kube-proxy
  lxc exec ${instance} -- mkdir -p /var/lib/kubernetes
  lxc exec ${instance} -- mkdir -p /var/run/kubernetes
  lxc exec ${instance} -- mkdir -p /etc/containerd/
done
}
```

Install the worker binaries:

```
{
  sudo mv runsc-50c283b9f56bb7200938d9e207355f05f79f0d17 runsc
  sudo mv runc.amd64 runc
  chmod +x kubectl kube-proxy kubelet runc runsc

  for instance in worker-0 worker-1 worker-2; do
    lxc file push kubectl ${instance}/usr/local/bin/
    lxc file push kube-proxy ${instance}/usr/local/bin/
    lxc file push kubelet ${instance}/usr/local/bin/
    lxc file push runc ${instance}/usr/local/bin/
    lxc file push runsc ${instance}/usr/local/bin/

    lxc file push crictl-v1.21.0-linux-amd64.tar.gz ${instance}/home/ubuntu/
    lxc file push cni-plugins-linux-amd64-v0.9.1.tgz ${instance}/home/ubuntu/
    lxc file push containerd-1.4.4-linux-amd64.tar.gz ${instance}/home/ubuntu/

    lxc exec ${instance} -- tar -xvf /home/ubuntu/crictl-v1.21.0-linux-amd64.tar.gz -C /usr/local/bin/
    lxc exec ${instance} -- tar -xvf /home/ubuntu/cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
    lxc exec ${instance} -- tar -xvf /home/ubuntu/containerd-1.4.4-linux-amd64.tar.gz -C /
  done
}
```

### Configure CNI Networking

Create the `bridge` network configuration file:

```
{
for instance in 0 1 2; do

POD_CIDR=10.1.1${instance}.0/24

cat <<EOF | tee 10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

lxc file push 10-bridge.conf worker-${instance}/etc/cni/net.d/

done
}
```

Create the `loopback` network configuration file:

```
cat <<EOF | tee 99-loopback.conf
{
    "cniVersion": "0.4.0",
    "type": "loopback"
}
EOF
```

### Configure containerd

Create the `containerd` configuration file:

```
cat << EOF | tee config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
    [plugins.cri.containerd.gvisor]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

> Untrusted workloads will be run using the gVisor (runsc) runtime.

Create the `containerd.service` systemd unit file:

```
cat <<EOF | tee containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubelet

Create the `kubelet-config.yaml` configuration file:

```
for instance in 0 1 2; do

POD_CIDR=10.1.${instance}.0/16

cat <<EOF | tee kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/worker-${instance}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/worker-${instance}-key.pem"
EOF

lxc file push kubelet-config.yaml worker-${instance}/var/lib/kubelet/

lxc file push worker-${instance}-key.pem  worker-${instance}/var/lib/kubelet/
lxc file push worker-${instance}.pem worker-${instance}/var/lib/kubelet/
lxc file push worker-${instance}.kubeconfig worker-${instance}/var/lib/kubelet/kubeconfig
lxc file push ca.pem worker-${instance}/var/lib/kubernetes/

done
```

> The `resolvConf` configuration is used to avoid loops when using CoreDNS for service discovery on systems running `systemd-resolved`.

Create the `kubelet.service` systemd unit file:

```
cat <<EOF | tee kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --fail-swap-on=false \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy

```
for instance in worker-0 worker-1 worker-2; do
  lxc file push kube-proxy.kubeconfig ${instance}/var/lib/kube-proxy/kubeconfig
done
```

Create the `kube-proxy-config.yaml` configuration file:

```
cat <<EOF | tee kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
conntrack:
  max: 0
  maxPerCore: 0
EOF
```

Create the `kube-proxy.service` systemd unit file:

```
cat <<EOF | tee kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Copy all the configuration files to all workers

```
for instance in worker-0 worker-1 worker-2; do
    lxc file push 99-loopback.conf ${instance}/etc/cni/net.d/
    lxc file push config.toml ${instance}/etc/containerd/
    lxc file push containerd.service ${instance}/etc/systemd/system/
    lxc file push kubelet.service ${instance}/etc/systemd/system/
    lxc file push kube-proxy-config.yaml ${instance}/var/lib/kube-proxy/
    lxc file push kube-proxy.service ${instance}/etc/systemd/system/
done
```

### Start the Worker Services

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- systemctl daemon-reload
  lxc exec ${instance} -- systemctl enable containerd kubelet kube-proxy
  lxc exec ${instance} -- systemctl start containerd kubelet kube-proxy

done
}
```

## SWAP issues

If your nodes failed to start (check the journalctl in one of the workers), there is a good chance that Kubelet is failing because swap is active. One way to fix this is to disable swap in your main server, not the container, with the command:

```
sudo swapoff -a
```

You need at least 8GB of memory to run everything without Swap with some performance. This Lab was tested in a machine with 8GB of ram.

Note: There is hack that needs to be done on all worker nodes, ensure this is in place when you restart the nodes

```
ln -s /dev/console /dev/kmsg

```

## Recommendation

Have a handy shell script that you will run every time when you restart worker nodes

```
{
for instance in worker-0 worker-1 worker-2; do
  lxc exec ${instance} -- ln -s /dev/console /dev/kmsg
done
}
```

## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.

List the registered Kubernetes nodes:

```
kubectl get nodes --kubeconfig admin.kubeconfig
```

> output

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   35s   v1.22.3
worker-1   Ready    <none>   36s   v1.22.3
worker-2   Ready    <none>   36s   v1.22.3
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)

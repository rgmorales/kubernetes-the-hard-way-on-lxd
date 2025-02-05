# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `cfssl` and `cfssljson` from the [cfssl repository](https://pkg.cfssl.org):

If you are using ARM64, you may have to use your workstation/mac to create certificate as cfssl only support AMD64 based architecture.

### OS X


```
brew install cfssl
```

### Linux
Get the latest release from below link 


https://github.com/cloudflare/cfssl/releases


### Verification

Verify `cfssl` version 1.6.0 or higher is installed:

```
cfssl version
```

> output

```
Version: 1.6.1
Runtime: go1.17.2
```

> The cfssljson command line utility does not provide a way to print its version.

## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

### OS X

```
brew install kubectl
```
or 

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.22.3/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.22.3/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Verification

Verify `kubectl` version v1.22.3 or higher is installed:

```
kubectl version --client
```

Next: [Provisioning Compute Resources](03-compute-resources.md)

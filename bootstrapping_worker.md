# Bootstraping the K8s Worker Nodes

In this lab exercise we will be setting up the worker nodes for the Kubernetes cluster. We will be installing the following components on the controller nodes:
* runc: CLI tool for spawning and running containers according to the OCI specification.
* gVisor: User-space kernel that implements a substantial portion of the Linux system surface alongwith an Open Container Initiative(OCI) runtime called ```runsc```.
* Container Networking Interface: ```cni``` is a which provides networking for linux containers.
* containerd: Industry standard container runtime.
* kubelet
* kube-proxy

All these should be installed on all the worker nodes.

> Note: Remember ```tmux```. Also keep the CIDR range of the subnet handy.

## Provisioning a Kubernetes Worker Node

### Install OS dependencies
Execute the following command:
``` bash
sudo apt install -y socat conntrack ipset
```
```socat``` enables support for the ```kubectl port-forward``` command.

### Install worker binaries
#### Install _circtl_ and _critest_
Install ```crictl```:
```bash
VERSION="v1.15.0" 
curl https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz --output /tmp/crictl-linux.tar.gz
sudo tar zxvf /tmp/crictl-linux.tar.gz -C /usr/local/bin
rm -f /tmp/crictl-linux.tar.gz
```

Install ```critest```:
``` bash
VERSION="v1.15.0"
curl https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/critest-$VERSION-linux-amd64.tar.gz --output /tmp/critest.tar.gz
sudo tar zxvf /tmp/critest.tar.gz -C /usr/local/bin
rm -f /tmp/critest.tar.gz
```

#### Install _runsc_
_runsc_ is used for container management and published by _gVisor_.
``` bash
curl https://storage.googleapis.com/gvisor/releases/nightly/latest/runsc
chmod +x runsc
cp runsc /usr/local/bin/
```

#### Install _runc_
_runc_ is a CLI tool for spawning and running containers according to the OCI specification.
``` bash
curl https://github.com/opencontainers/runc/releases/download/v1.0.0-rc8/runc.amd64 --output runc
chmod +x runc
cp runc /usr/local/bin/
```

#### Install _cni-plugins_
``` bash
curl https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz --output cni-plugins.tgz
tar zxvf cni-plugins.tgz

```

#### Install _containerd_
```bash
curl 
```

#### Install _kubectl_, _kube-proxy_ and _kubelet_
```bash
curl https://dl.k8s.io/v1.15.0/kubernetes-node-linux-amd64.tar.gz --output /tmp/kubernetes-node-linux-amd64.tar.gz
tar -xvf /tmp/kubernetes-node-linux-amd64.tar.gz
cp /tmp/kubernetes/node/bin/kubectl /usr/local/bin/
cp /tmp/kubernetes/node/bin/kube-proxy /usr/local/bin/
cp /tmp/kubernetes/node/bin/kubelet /usr/local/bin/
rm -rvf /tmp/kubernetes-node-linux-amd64.tar.gz /tmp/kubernetes
```

## Configuring CNI Networking
We need the CIDR range of the current node, this you could get from the azure console. Create the ```bridge``` network configuration file ```/etc/cni/net.d/10-bridge.conf```:
``` json
{
    "cniVersion": "0.2.0",
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
```
We will also need a ```loopback``` network configuration file:
``` json
{
    "cniVersion": "0.2.0",
    "name": "lo",
    "type": "loopback"
}
```

## Configuring _containerd_
_containerd_ manages the complete container lifecycle of the host system, from image transfer and storage to container execution and supervision to low-level storage to network attachments and beyond. The diagram below is the architecture of _containerd_.
![containerd](https://containerd.io/img/architecture.png)

### Configuration
Create the ```containerd``` configuration file:
``` bash
sudo mkdir -p /etc/containerd/
```
Create a ```containerd``` config file with the following configuration:
```
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
```

### Register service
Create a service for ```containerd``` by creating the ```containerd.service``` in ```/etc/systemd/system/containerd.service```:
```
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd

Delegate=yes
KillMode=process
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity

[Install]
WantedBy=multi-user.target
```

## Configuring _kubelet_

### Configuration
Copy the host key, kubeconfig and ca key which had been generated earlier
```
sudo mv gb-k8s-w-0-key.pem gb-k8s-w-0.pem /var/lib/kubelet/
sudo mv gb-k8s-w-0.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

Create the ```/var/lib/kubelet/kubelet-config.yaml``` configuration file:
```
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
tlsCertFile: "/var/lib/kubelet/gb-k8s-w-0.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/gb-k8s-w-0-key.pem"
```

> The ```resolvConf``` configuration is used to avoid loops when using CoreDNS for service discovery on systems running ```systemd-resolved```.

### Register service
Create ```kubelet.service``` systemd unit file in ```/etc/systemd/system/```:
```
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
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Configuring _kube-proxy_

### Configuration
Copy the config file into ```/var/lib/kube-proxy/kubeconfig```:
``` bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the ```kube-proxy-config.yaml``` file in ```/var/lib/kube-proxy/```:
``` yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
```

### Register service
Create the ```/etc/systemd/system/kube-proxy.service``` file:
```
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
```

## Start Services
Start all the services: ```containerd```, ```kubelet```, ```kube-proxy```
``` bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

> Note: All these commands are to be run on each worker node: ```gb-k8s-w-0```, ```gb-k8s-w-1```, ```gb-k8s-w-2```

## Verification
Login into the controller node and execute the following command:
``` bash
kubectl get nodes
```

The output should be:
```
NAME         STATUS    AGE       VERSION
gb-k8s-w-0   Ready    <none>   32s   v1.15.0
gb-k8s-w-1   Ready    <none>   32s   v1.15.0
gb-k8s-w-2   Ready    <none>   37s   v1.15.0
```
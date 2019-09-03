# Bootstraping the K8s Control Plane
In this lab exercise we will be setting up the control plane across the three controller nodes and configure it for high availability. We will be installing the following components on the controller nodes:
* API Server
* Scheduler
* Controller Manager
We will also be creating an external load balancer for Kubernetes API Servers so that kubectl can be executed on the local system and also enable display of the kubernetes admin console.

>Note:
> All the following commands needs to be executed on each controller node

>tmux will be helpful here as with the previous exercise

## Connect to a control-plane node
SSH into the the control-plane node
``` bash
ssh username@gb-k8s-0-ip -p
```

## Provision Control Plane
First we need to create the configuration directory
``` bash
sudo mkdir -p /etc/kubernetes/config
```

### Download the API Server, Controller Manager, Scheduler and Kubectl
You can download all the binaries by the following command:
``` bash
curl -X GET https://dl.k8s.io/v1.15.0/kubernetes-server-linux-amd64.tar.gz --output k8s-server.tar.gz

tar -xf k8s-server.tar.gz
```

Update the permissions on the files to be executable
``` bash
cd kubernetes/server/bin
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
```

This would create a kubernetes folder. We need to copy the required files from ```kubernetes/server/bin``` into ```/usr/local/bin```.

```bash
sudo cp kubernetes/server/bin/kube-apiserver /usr/local/bin
sudo cp kubernetes/server/bin/kube-controller-manager /usr/local/bin
sudo cp kubernetes/server/bin/kube-scheduler /usr/local/bin
sudo cp kubernetes/server/bin/kubectl /usr/local/bin
```

### Configure the API Server
Create the folder to store the certificates and encryption configuration files.
``` bash
sudo mkdir -p /var/lib/kubernetes/
sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem \
    service-account.pem encryption-config.yaml /var/lib/kubernetes
```

### Environment Setup
Set an environment variable ```INTERNAL_IP```.
``` bash
INTERNAL_IP=<azure-ip-address>
```

### Configure API Server as a service
Create a file ```kube-apiserver.service``` in ```/etc/systemd/system```.

``` bash
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
  --etcd-servers=https://<gb-k8s-cp-0>:2379,https://<gb-k8s-cp-1>:2379,https://<gb-k8s-cp-2>:2379 \\
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
```

```<gb-k8s-cp-0>``` is the ip address of the control-panel nodes.

### Configure the Controller Manager
Copy the controller manager ```kube-controller-manager.kubeconfig``` into ```/var/lib/kubernetes```.
``` bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

### Configure Controller Manager as a service
Create a file ```kube-controller-manager.service``` in ```/etc/systemd/system```.
``` bash
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=<subnet-cidr> \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```<subnet-cidr>``` is the CIDR range of the subnet.

### Configure the Scheduler
Create a ```kube-scheduler.yaml``` configuration file:
``` yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
    kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
    leaderElect: true
```

Copy the kube-scheduler kubeconfig into ```/var/lib/kubernetes```.
``` bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

### Configure the Scheduler as a service
Create a file ```kube-scheduler.service``` in ```/etc/systemd/system```.
```
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
```

### Start the Controller Services
Execute the following commands to start the services
``` bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-ccontroller-manager kube-scheduler
```
>Note:
> It takes around 10 seconds to fully initialize the services

### Enable HTTP Health Checks

### Verification of the components

## RBAC for Kubelet Authorization

## Kubernetes Frontend Load Balancer

### Provision a Network Load Balancer

### Verification
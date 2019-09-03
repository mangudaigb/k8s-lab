# Generating K8s Configuration Files for Authentication
In this exercise we will be generating kubeconfig files to organize information about clusters, users, namespaces and authentication mechanisms. So, what is a kubeconfig file? The answer to it is quite simple, any file which is used to configure access to the Kubernetes cluster is called a kubeconfig file. This is a generic name and does not in any way mean that there exists a file named kubeconfig.
The default location of the kubeconfigs are $HOME/.kube directory. Another way in which the kubeconfigs are extracted is KUBECONFIG environment variable.

## Client Authentication Configs
We will be generating kubeconfigs for controller manager, kubelet, kube-proxy and scheduler clients and the admin user.

### Kubernetes public IP address
Each kubeconfig requires a Kubernetes API server, so for high availibility we will be using the public IP address we created in the [Provisioning Compute Resources](provision_compute_resources.md) section.
``` bash
export K8S_PUBLIC_IP=""
```

### kubelet kubeconfigs
When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes Node Authorizer.
``` bash
for instance in gb-k8s-w-0 gb-k8s-w-1 gb-k8s-w-2; do
  kubectl config set-cluster gb-k8s \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${K8S_PUBLIC_IP}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=gb-k8s \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
This generates the following files:
```
gb-k8s-w-0.kubeconfig
gb-k8s-w-1.kubeconfig
gb-k8s-w-2.kubeconfig
```

### kubbe-proxy kubeconfig
We will be generating kubeconfigs for the kube-proxy pods.
``` bash
kubectl config set-cluster gb-k8s \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${K8S_PUBLIC_IP}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
--client-certificate=kube-proxy.pem \
--client-key=kube-proxy-key.pem \
--embed-certs=true \
--kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
--cluster=gb-k8s \
--user=system:kube-proxy \
--kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
This generates ``` kube-proxy.kubeconfig``` file.

### kube-controller-manager kubeconfig
Generate kubeconfig for the controller-manager:
``` bash
kubectl config set-cluster gb-k8s \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=gb-k8s \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```
This generates ```kube-controller-manager.kubeconfig``` file.

### kube-scheduler kubeconfig
No idea what to write, you all know :)
``` bash
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```
This generates ```kube-scheduler.kubeconfig``` file.

### Admin user kubeconfig
Damn again, OK! superman kicks batman's ass in every dimension and every parallel universe.
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```
This generates ```admin.kubeconfig``` file.

## Copy the kubeconfigs
### kubelets
Copy the kubelet configs to the appropriate worker nodes.
``` bash
scp -r /path/to/kubeconfig username@worker-ip-address:/path/to/destination
```
### kube-proxy
Copy the kube-proxy configs to all the worker nodes.
```
scp /path/to/kube-proxy username@worker-ip-address:/path/to/destination
```
### kube-controller-manager
Copy the kube-controller-manager configs to all the controller nodes.
```
scp /path/to/kube-controller-manager username@controller-ip-address:/path/to/destination
```
### kube-scheduler
Copy the kube-controller-manager configs to all the controller nodes.
```
scp /path/to/kube-scheduler username@controller-ip-address:/path/to/destination
```
### admin user
Copy the admin user configs to all the controller nodes.
```
scp /path/to/admin username@controller-ip-address:/path/to/destination
```
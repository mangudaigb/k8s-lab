# Configuring _kubectl_ for Remote Access

In this exercise we will be generating a kubeconfig for the ```kubectl``` command utility based on the admin user credentials.
> The commands will be run from the same directory used to generate the admin client certificates.

## Admin Kubernetes Configuration File
Generate a kubeconfig suitable for authenticating as the ```admin``` user:
``` bash
LOAD_BALANCER_IP=<ip-of-load-balancer>
kubectl config set-cluster gb-k8s --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://$(LOAD_BALANCER_IP):6443
```

Set the credentials for the admin user:
``` bash
kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem
```

Set the security context for the admin. Security context defines the privileges and access control settings for a Pod/Container.
``` bash
kubectl config set-context gb-k8s --cluster=gb-k8s \
    --user=admin
```

Activate the security context for _kubectl_ by:
``` bash
kubectl config use-context gb-k8s
```

## Verification
Check the health of the remote kubernetes cluster with:
``` bash
kubectl get componentstatuses
```
> Expected output
```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
```

You can also list all the nodes with:
```
kubectl get nodes
```

# Deploying the DNS Cluster Add-On

Kubernetes DNS schedules a DNS Pod and Service on the cluster, and configures the kubelets to tell individual containers to use the DNS Service's IP to resolve DNS names. Every service defined in the cluster, including the DNS service, is assigned a DNS name. 
> Note: If you have a service _Sa_ in the k8s namespace _Na_; A pod running in namespace _Nb_ can access the service by doing a DNS query for _Sa.Na_.

>Note: In general, kubernetes supports A records, CNAME and SRV records.

## Installing 
There are two flavors of DNS available: _CoreDNS_ and _Kube-dns_. We will be using _CoreDNS_ in this exercise. Versions later to 1.13 have CoreDNS installed by default.


## Verification
We will create a ```busybox``` deployment:
``` bash
kubectl run --generator=run-pod/v1 busybox --image=busybox:1.28 --command -- sleep 3600
```

List the pod created by the ```busybox``` deployment:
```
kubectl get pods -l run=busybox
```

We will be getting an output like:
```
NAME                      READY   STATUS    RESTARTS   AGE
busybox                   1/1     Running   0          10s
```

Extract the full name of the ```busybox``` pod:
```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Now we will query the DNS for the ```kubernetes``` service inside the ```busybox``` pod:
```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```
You will be getting an output like:
```
Server:     10.32.0.10
Address:    10.32.0.10 kube-dns.kubesystem.svc.cluster.local

Name:       kubernetes
Address 1:  10.32.0.1 kubernetes.default.svc.cluster.local
```


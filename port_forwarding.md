# PORT FORWARDING In Kubernetes
_Port Forwarding_ or _port mapping_ is an application of NAT that redirects a communication request from one address and port number combination to another at the network layer 6.

You can use _port forwarding_ in the kubernetes cluster to connect to external services. This helps majorly in debugging scenarios. Imagine you have a MySQL database running on 3306, you can forward the port 7000 on your local system to 3306 of the pod, and you have setup your local workstation to debug the database that is runnign in kubernetes.

## Usecases
### Pod
Listen on ports 5000 and 6000 locally, forwarding data to/from ports 5000 and 6000 in the pod.
```
kubectl port-forward pod/mypod 5000 6000
```

Listen on port 8888 locally, forwarding to 5000 in the pod.
```
kubectl port-forward pod/mypod 8888:5000
```

Listen on port 8888 on all addresses, forwarding to 5000 in the pod.
```
kubectl port-forward --address 0.0.0.0 pod/mypod 8888:5000
```

Listen on port 8888 on localhost and selected IP, forwarding to 5000 in the pod.
```
kubectl port-forward --address localhost,10.19.21.23 pod/mypod 8888:5000
```

Listen on a random port locally, forwarding to 5000 in the pod.
```
kubectl port-forward pod/mypod :5000
```

### Deployment
Listen on ports 5000 and 6000 locally, forwarding data to/from ports 5000 and 6000 in a pod selected by the deployment.
```
kubectl port-forward deployment/mydeployment 5000 6000
```

### Service
Listen on ports 5000 and 6000 locally, forwarding data to/from ports 5000 and 6000 in a pod selected by the service.
```
kubectl port-forward service/myservice 5000 6000
```

# Provisioning Pod Network Routes

Pods running on a node are assigned an IP address from the node's Pod CIDR range. As we have not configured network routes, these pods can only communicate with other pods on the same node but not with pods which are on other nodes. We will be configuring the network routes in this exercise so that all pods can communicate with each other irrespective of the node in which they are located.

Some important key points of the Kubernetes network model:
1. Pods on a node can communicate with all pods on all nodes without NAT.
2. Agents on a node can communicate with all pods on that node.
3. Pods in the host network of a node can communicate with all pods on all nodes without NAT.

> Containers on a pod share the network namespace, hence they can communicate with other containers on the same pod with ```localhost```.

## Routing Table
Get the internal IP address and Pod CIDR range for each worker instance:

>output will be soemthing like
> ```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes
Create network routes for the worker instance:
```
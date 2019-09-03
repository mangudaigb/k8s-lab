# Bootstraping the _etcd_ Cluster
Kubernetes components are stateless and the state data is stored only in the _etcd_. _etcd_ is nothing but a distributed key-value store which provides high reliability and strong consistency. Imagine zookeeper but with no extra baggage at the cost of some functionalities.

> The commands in this exercise are to be run in every controller instance. So ssh into your controller and fireaway. _tmux_ here will be very helpful.

## Installing _etcd_
Download the official etcd release binaries from the etcd Github project:
``` bash
ETCD_VER=v3.3.15
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
rm -f /tmp/etcd-${ETCD_VER}-darwin-amd64.zip
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${GITHUB_URL}/${ETCD_VER}/etcd-${ETCD_VER}-darwin-amd64.zip -o /tmp/etcd-${ETCD_VER}-darwin-amd64.zip
unzip /tmp/etcd-${ETCD_VER}-darwin-amd64.zip -d /tmp && rm -f /tmp/etcd-${ETCD_VER}-darwin-amd64.zip
mv /tmp/etcd-${ETCD_VER}-darwin-amd64/* /tmp/etcd-download-test && rm -rf mv /tmp/etcd-${ETCD_VER}-darwin-amd64
```

Check the etcd version:
``` bash
/tmp/etcd-download-test/etcd --version
```
Check the etcdctl(etcd client) version:
``` bash
ETCDCTL_API=3 /tmp/etcd-download-test/etcdctl version
```

## Start and verify the _etcd_
Copy the etcd and etcdctl binary to your _/usr/local/bin_ folder and start the _etcd_ server with the following command:
``` bash
etcd
```
That's it!

>Note:
>Golang produces full executables with no external dependencies, so copying the executable will always work; does not matter where you copy.

To verify if the server is correctly setup, we can run the following commands to do some sample read and writes:
``` bash
ETCDCTL_API=3 etcdctl --endpoints=localhost:2379 put gb tesco
ETCDCTL_API=3 etcdctl --endpoints=localhost:2379 get gb
```
The second command should return ```tesco``` as the output. Not shutdown _etcd_ by killing the process
``` bash
ps -ef | grep etcd
```
Get the process id from the above command and then ```kill <process-id>```.

## Configuring _etcd_ environment
Retrieve the instance internal IP address as this will be used to serve client requests and to communicate with _etcd_ cluster peers. Each _etcd_ member in the cluster must have an unique name in the etcd cluster. As we have uniquely defined our nodes with unique names(remember gb-k8s-cp-0,gb-k8s-cp-1,gb-k8s-cp-2) we just need to set the _ETCD_NAME_ environment variable to the hostname.
``` bash
export ETCD_NAME=$(hostname -s)
```

## Registering _etcd_ as service
We would not like _etcd_ to be manually started whenever the node goes down, so we will be registering _etcd_ as a service.

### Create the etcd.service file
Create a file with the name ```etcd.service``` in ```/etc/systemd/system``` folder. 
The file should contain the following text:
``` service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://<controller-0-ip>:2380,controller-1=https://<controller-1-ip>:2380,controller-2=https://<controller-2-ip>:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
The part in ```<controller-{number}-ip>``` is to be replaced by the actual ips of the nodes.
>Note:
> Remember we had generated and copied all the certificates in a previous exercise.

### Reload system configuration
Execute the following commands in order:
``` bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```
The last command starts the etcd server with the new configurations we have provided.

## Verification
We will be listing all the _etcd_ cluster members after the above steps have been run on all the control-plane nodes. 
The following command is run on any one of the control-plane nodes:
``` bash
sudo etcdctl member list --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/etcd/ca.pem \
    --cert=/etc/etcd/kubernetes.pem \
    --key=/etc/etcd/kubernetes-key.pem
```
And you will get some output like:
``` bash
3a57933972cb5131, started, gb-k8s-cp-0, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, gb-k8s-cp-1, https://10.240.0.10:2380, https://10.240.0.10:2379
ffed16798470cab5, started, gb-k8s-cp-2, https://10.240.0.11:2380, https://10.240.0.11:2379
```
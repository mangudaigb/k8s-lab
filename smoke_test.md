# Smoke Test
We will be testing our cluster on various aspects provided by Kubernetes.

## Data Encryption
In this part we will be verifying if the data at rest is encrypted. 
> Note:
> Data encryption generally belongs to two categories: 1. Data at rest and 2. Data in motion. Encrypted data when stored in some form of persistence storage, can be database or file system, is considered data at rest; while encryption protocols like TLS fall in the data in motion category.

Lets a create a generic secret:
```
kubectl create secret generic gb-k8s --from-literal="key=data"
```

Print the hexdump of the ```gb-k8s``` secret stored in etcd in one of the controllers:
```
PUBLIC_IP_ADDRESS=$()
ssh kuberoot@${PUBLIC_IP_ADDRESS} \
    "ETCDCTL_API=3 etcdctl get /registry/secrets/default/gb-k8s | hexdump -C"
```

We will have someoutput like:
```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/gb-k8s|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |.k8s:enc:aescbc|
00000030  3a 76 31 3a 6b 65 79 31  3a 65 c3 db a8 fb ae 9b  |:v1:key1:e......|
00000040  f9 09 59 0b 12 fa 4f 5d  4c 6c c5 35 28 d8 72 08  |..Y...O]Ll.5(.r.|
00000050  f7 9e 4b 0a 6e 1d 6b 27  8f d2 7f 36 2b 11 6b 61  |..K.n.k'...6+.ka|
00000060  53 6a a7 24 56 e2 19 ee  e7 04 94 ee b3 9c d3 c3  |Sj.$V...........|
00000070  68 b5 b8 51 8b 01 4e d9  f0 ce 40 9a 73 5c 10 28  |h..Q..N...@.s\.(|
00000080  18 bc ff 3a 51 4d bc 0c  6d 27 97 5c c6 bd a2 35  |...:QM..m'.\...5|
00000090  88 18 56 16 c7 10 12 a1  e2 cf c5 62 6c 50 7e 67  |..V........blP~g|
000000a0  89 0c 42 56 73 69 48 bf  24 5e 91 91 56 2d 64 2f  |..BVsiH.$^..V-d/|
000000b0  3a 35 b9 c9 08 41 d6 95  62 e8 1b 35 80 c9 8e 74  |:5...A..b..5...t|
000000c0  79 34 bc 5b 7c 68 cd 0c  bc 11 21 c0 48 bc 92 a6  |y4.[|h....!.H...|
000000d0  2f b5 ef 18 5c f1 00 16  19 22 e8 9c c1 8c 3c 35  |/...\...."....<5|
000000e0  fa b3 87 51 85 bf f0 cd  0e 0a                    |...Q......|
000000f0
```

The etcd key should be prefixed with ```k8s:enc:aescbc:v1:key1```, which indicates the ```aescbc``` provider was used to encrypt the data with the ```key1``` encryption key.

## Deployments


## Port Forwarding
In this section we you will verify the ability to access applications remotely using _[port forwarding](port_forwarding.md)_. 

All the following commands produce the same output:
```
kubectl port-forward redis-master-765d459796-258hz 7000:6379
```
or
```
kubectl port-forward deployment/redis-master 7000:6379
```
or
```
kubectl port-forward rs/redis-master 7000:6379
```
or
```
kubectl port-forward svc/redis-master 7000:6379
```

## Logs
Retrive the container logs with the following command:
```
kubectl logs $POD_NAME
```

## Exec
Verify the ability to execute commands in a container.
```
kubectl exec -ti $POD_NAME -- nginx -v
```

## Services
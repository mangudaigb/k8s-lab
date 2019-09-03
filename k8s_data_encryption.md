# Generating the Data Encryption Config and Key
These configuration are used by Kubernetes to encrypt data at rest. This data includes the cluster state, application configurations and secrets etc.

A sample configuration for data at rest looks like this:
``` yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - identity: {}
    - aesgcm:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
        - name: key2
          secret: dGhpcyBpcyBwYXNzd29yZA==
    - aescbc:
        keys:
        - name: key1
          secret: c2VjcmV0IGlzIHNlY3VyZQ==
        - name: key2
          secret: dGhpcyBpcyBwYXNzd29yZA==
    - secretbox:
        keys:
        - name: key1
          secret: YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY=
``` 
The _resources_ item constitutes a complete configuration. The resources.resources field is an array of Kubernetes resource names(**resource** or **resource.group**) that should be encrypted. The _providers_ array is an ordered list of the possible encryption providers. Care should be taken that _identity_ or _aescbc_ may be provided, but not both at the same time. If any resource is not readable via the encryption config because the keys were changed, the only recourse is to delete that key from the underlying etcd directly.

## Generate a Secret key
This key should be a base64 encoded secret.
``` bash
openssl rand -base64 32
```
On my system it gave an output of ``` zc0udU1vPraHXsP7np01eNgJODZ0OKrMDDKUSFgEvD4= ```. Everyone should be getting a different value.

## Encrypting config file
Create the encryption config file as ```encryption-config.yaml```:
``` yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: zc0udU1vPraHXsP7np01eNgJODZ0OKrMDDKUSFgEvD4=
    - identity: {}
```
Copy this file to all the controller instances.
``` bash
for instance in gb-k8s-w-0 gb-k8s-w-1 gb-k8s-w-2; do
    scp /path/to/configfile username@controller-instance-ip:~/
done
```

## Configure Kubernetes to use this key
We will be using this config file when we start our pods.

Note:
> This config file contains sensitive keys which can decrypt content in etcd, so proper restrict permissions need to be applied.
# Generating TLS Certificates
Transport Layer Security(TLS) is a cryptographic protocol designed to provide security and data integrity for communications in a computer network. 
Once the client and server have agreed to use TLS, a stateful connection is first established using a handshaking procedure. Generally this handshake starts with a "STARTTLS" message from the client to the server.
The handshake is implemented using an asymmetric cipher and results in determination of cipher settings understood by both parties involved and may also include a session-specific symmetric cipher for further communication. The reason why symmetric cipher is used after the handshake is because of the computational cost in the encryption and decryption of an asymmetric cipher. 
To enumerate the steps of the handshake:
1. The client send a STARTTLS message to the server.
2. On success, the client sends a list of supported cipher suites and hash functions.
3. The server picks up a cipher suite and a hash function that it also supports and notifies the client.
4. The server then provides its digital certificate.
5. The client verifies the identity of the server by verifying certificate against a Certificate Authority.
6. A symmetric key is decided for further communication. Generally, Diffie-Hellman Key Exchange is used here, but there are other protocols which support this too.
This concludes the handshake and begins the secured connection until the connection closes. Currently, the most popularly supported protocol is TLS 1.2 but TLS 1.3 is the latest one.

The reason why we are talking to TLS is, we would need to set up secure communication among the kubernetes nodes. We do not want anyone evesdropping on our server. In this section we will be generating a CA, generating certificates for the kubernetes admin user, kubelet, controller manager, kube proxy, scheduler client, kubernetes API server and the service account.

## What is a Certificate Authority
A certificate authority is an entity which issues certificates for second party entities. The CA is trusted by both the servers and the clients, and thus acts as a "trusted third party" for both the server and the client. The format of the certificates is specified by the X.509 standard.
To give an example, when we browse a https enabled website on the browser(like facebook or google), the browser knows that it is the correct site(and displays a lock symbol in firefox) by verifying the certificate provided by the website against the root certificates stored in the browser's datastore. Yupp, the certificate generated by a CA is also called the root certificate. Now you might ask who installs these certificates on the browser; well, all the major CA certificates come packaged when you install a browser. If you goto **about:preferences** in firefox and click on the _Privacy and Security_ tab and scroll down to the bottom of the page, you will see a button _**View Certificate**_. Click on the button and you would see all the certificates installed on your browser.

As CAs charge a lot of money generating certificates, we will be generating our own CA certificates and forcing the nodes to trust them. :)

## What is a Certificate
A certificate is nothing but a string of data which is used to verify the ownership of the public key presented by a subject. A certificate includes the public key, some information about the identity of the _subject_ and the digital signature of the Certicate Authority. 

## Kubernetes and TLS Client Authentication
In two way TLS, after the client has verified the server's certificate, it presents its own certificate to the server for verification. Kubernetes uses this "TLS Client Authentication"(its also refered as client TLS authentication) to communicate between the pods except the nodeport services. In the Kubernetes cluster, the components on the worker nodes-kubelet and lube proxy- communicate with the master components, specifically the api server using this TLS CLient Authentication. This enables a passwordless authentication mechanism between the nodes. We will be generating certificates for the different pods in the next section.

### Folder structure
Create the following folder structure and keep the generates certificates and keys in their rrespective folders:
```
.
+-- ca
|   +-- private.key
|   +-- csr
+-- admin
|   +-- 
+-- kubelet
|
+-- controller-manager
|
+-- kube-proxy
|
+-- scheduler
|
+-- api-server
|
+-- service-account
```
Use the following bash command to create these directories
``` bash
mkdir ca admin kubelet controller-manager kube-proxy scheduler api-server service-account
```

### Certificate Authority
We will be first generating the CA certificate(in the ca folder but the code will be executed against the current folder), which will then be used to generate certificates for the pods. 
First, generate the CA private key
``` bash
umask 077; openssl genrsa -out ca/private.pem -des3 2048
```
Generate a X.509 certificate for the CA
``` bash
umask 077; openssl req -new -x509 -nodes -key ca/private.pem -sha256 -days 365 -out ca/cacert.crt
```
You will be prompted for a password, which is the password you had provided in the earlier step. Along with the password, you will be needing to enter some more details in an interactive way:
``` bash
Country Name (2 letter code) []:IN
State or Province Name (full name) []:KA
Locality Name (eg, city) []:BENGALURU
Organization Name (eg, company) []:TESCO
Organizational Unit Name (eg, section) []:DCP
Common Name (eg, fully qualified host name) []:K8S
Email Address []:gb@gb.com
```
This would create a certificate named ```cacert.crt``` and you are done with CA.

>Note:
Ideally you would be creating an intermediate certificate and then generating certificates for the nodes from this intermediate certificates, but you can skip this "good practice" step for this exercise.

### Admin Client Certificates
We will be generating CA signed admin certificates for the admin pod of kubernetes. First generate a private key for the admin node.

#### Create a private key
``` bash
mkdir admin
umask 077; openssl genrsa -out admin/private.key 2048
```

#### Create CSR Config file
Create a config file ```_admin/csr.conf_``` for the admin's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]

countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:masters

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(OU) (e.g. server FQDN or YOUR name)
commonName_default          = admin

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]

subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key admin/private.key -out admin/admin.csr -config admin/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and admin.csr.
```
umask 077; openssl x509 -req -in admin/admin.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out admin/server.crt -days 365 -extfile admin/csr.conf
```

### Kubelet Client Certificates
Kubernetes uses Node Authorizer to authorize API requests made by kubelets, although RBAC based authorization is also supported.To be authorized by the Node Authorizer, kubelets must use a credential that identifies them as being in the ```system:nodes``` group, with a username of ```system:node:<node-name>```. You will be creating a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

>Note: The following commands will be executed for each worker node.

#### Create a private key
Generate a private key for the worker node:
``` bash
mkdir worker-0
umask 077; openssl genrsa -out worker-0/private.key 2048
```

#### Create a CSR Config file
Create a config file ```_worker-0/csr.conf_``` for the worker's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]
countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:nodes

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(CN) (eg. server FQDN or YOUR name)
commonName_default          = system:node:0

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]

subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key worker-0/private.key -out worker-0/worker-0.csr -config worker-0/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and worker-0.csr.
```
umask 077; openssl x509 -req -in worker-0/worker-0.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out worker-0/worker-0.crt -days 365 -extfile worker-0/csr.conf
```

### Controller Manager Client Certificates
You will be generating the kube-controller-manager private key and client certificate.

#### Create a private key
Generate a private key for the worker node:
``` bash
mkdir controller-manager
umask 077; openssl genrsa -out controller-manager/private.key 2048
```

#### Create a CSR Config file
Create a config file ```_controller-manager/csr.conf_``` for the worker's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]
countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:kube-controller-manager

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(CN) (eg. server FQDN or YOUR name)
commonName_default          = system:kube-controller-manager

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key controller-manager/private.key -out controller-manager/controller-manager.csr -config controller-manager/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and controller-manager.csr.
```
umask 077; openssl x509 -req -in controller-manager/controller-manager.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out controller-manager/controller-manager.crt -days 365 -extfile controller-manager/csr.conf
```

### Kube Proxy Client Certificates
You will be generating the kube-proxy private key and client certificate.

#### Create a private key
Generate a private key for the worker node:
``` bash
mkdir kube-proxy
umask 077; openssl genrsa -out kube-proxy/private.key 2048
```

#### Create a CSR Config file
Create a config file ```kube-proxy/csr.conf_``` for the node's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]
countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:node-proxier

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(CN) (eg. server FQDN or YOUR name)
commonName_default          = system:kube-proxy

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key kube-proxy/private.key -out kube-proxy/kube-proxy.csr -config kube-proxy/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and kube-proxy.csr.
```
umask 077; openssl x509 -req -in kube-proxy/kube-proxy.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out kube-proxy/kube-proxy.crt -days 365 -extfile kube-proxy/csr.conf
```

### Scheduler Client Certificates
You will be generating the kube-scheduler private key and client certificate.

#### Create a private key
Generate a private key for the worker node:
``` bash
mkdir kube-scheduler
umask 077; openssl genrsa -out kube-scheduler/private.key 2048
```

#### Create a CSR Config file
Create a config file ```_kube-scheduler/csr.conf_``` for the node's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]
countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:kube-scheduler

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(CN) (eg. server FQDN or YOUR name)
commonName_default          = system:kube-scheduler

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key kube-scheduler/private.key -out kube-scheduler/kube-scheduler.csr -config kube-scheduler/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and kube-scheduler.csr.
```
umask 077; openssl x509 -req -in kube-scheduler/kube-scheduler.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out kube-scheduler/kube-scheduler.crt -days 365 -extfile kube-scheduler/csr.conf
```

### Kubernetes API Server Certificates
You will be generating the kube-api private key and client certificate.

#### Create a private key
Generate a private key for the worker node:
``` bash
mkdir kube-api
umask 077; openssl genrsa -out kube-api/private.key 2048
```

#### Create a CSR Config file
Create a config file ```kube-api/csr.conf_``` for the node's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]
countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:k8s

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(CN) (eg. server FQDN or YOUR name)
commonName_default          = system:kube-api

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key kube-api/private.key -out kube-api/kube-api.csr -config kube-api/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and kube-api.csr.
```
umask 077; openssl x509 -req -in kube-api/kube-api.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out kube-api/kube-api.crt -days 365 -extfile kube-api/csr.conf
```

### Service Account Key Pair
You will be generating the kube-service private key and client certificate.

#### Create a private key
Generate a private key for the service node:
``` bash
mkdir kube-service
umask 077; openssl genrsa -out kube-service/private.key 2048
```

#### Create a CSR Config file
Create a config file ```kube-service/csr.conf_``` for the node's Certificate Signing Request(CSR) as shown below:
```
[ req ]
default_bits        = 2048
default_keyfile     = server-key.pem
distinguished_name  = subject
req_extensions      = req_ext
x509_extensions     = x509_ext
string_mask         = utf8only

[ subject ]
countryName                 = Country Name(C) (2 letter code)
countryName_default         = IN

stateOrProvinceName         = State or Province Name(S) (full name)
stateOrProvinceName_default = Karnataka

localityName                = Locality Name(L) (eg, city)
localityName_default        = Bengaluru

organizationName            = Organization Name(O) (eg, company)
organizationName_default    = system:k8s

organizationalUnitName      = Organizational Unit Name(OU) (eg. Department)
organizationalUnitName_default  = gb-k8s

commonName                  = Common Name(CN) (eg. server FQDN or YOUR name)
commonName_default          = system:service-accounts

emailAddress                = Email Address
emailAddress_default        = jibitesh.prasad@tesco.com

[ x509_ext ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid,issuer

basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
subjectAltName         = @alternate_names
nsComment              = "OpenSSL Generated Certificate"

[ req_ext ]
subjectKeyIdentifier = hash

basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1       = dev.dcp.com
```

#### Create a CSR
Create a CSR for the admin:
``` bash
umask 077; openssl req -new -key kube-service/private.key -out kube-service/kube-service.csr -config kube-service/csr.conf
```

#### Generate Certificate
Generate the server SSL certificate using the ca.key, ca.crt and kube-service.csr.
```
umask 077; openssl x509 -req -in kube-service/kube-service.csr -CA ca/cacert.crt -CAkey ca/private.pem -CAcreateserial -out kube-service/kube-service.crt -days 365 -extfile kube-service/csr.conf
```

## Distribute Client and Server Certificates
You will be copying the certificates necessary for setting up workers and the controllers.

### Worker Instances
Copy the ca.pem, ${instance}-key.pem, ${instance}.pem, kuberoot@${PUBLIC_IP_ADDRESS}

### Controller Instances
Copy the ca.pem, ca-key.pem, kubernetes-key.pem, kubernetes.pem, service-account-key.pem, service-account.pem, kuberoot@${PUBLIC_IP_ADDRESS}
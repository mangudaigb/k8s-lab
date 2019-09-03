# Install Tools on your system

You will be installing command line tools which will be helpful in installing and managing the kubernetes cluster.

# BREW
A package manager to install apps easily
``` bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```


# OPENSSL
You will be using _openssl_ to generate the various client certificates. Check if openssl is already isntalled on your system
```bash
openssl version -a
```
>Output:
``` bash
LibreSSL 2.6.5
built on: date not available
platform: information not available
options:  bn(64,64) rc4(16x,int) des(idx,cisc,16,int) blowfish(idx)
compiler: information not available
OPENSSLDIR: "/private/etc/ssl"
```

To install openssl if not installed:
```
brew install openssl
```

# Java
Install java compiler to build and deploy a _Hello World_ service in kubernetes.

# Kubectl
The ```kubectl``` command line utility is used to interact with the kubernetes API Server. To install execute the following command:
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl
```

Make the ```kubectl``` binary executable
```
chmod +x kubectl
```

Adding ```kubectl``` to the PATH variable:
```
sudo mv kubectl /usr/local/bin/kubectl
```
>Note:
Other way is to add ```kubectl``` to the PATH variable.

## Verify
Ensure the version installed is up-to-date:
```
kubectl version
```
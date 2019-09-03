# Provisioning Compute Resources
Kubernetes requires a set of machines to host the Control Plane and the worker nodes. The worker nodes run the various pods whereas the Control Plane is used for administrating the Kubernetes cluster. In this exercise we will be provisioning the required nodes in Azure. You can also use VMs but you would need a hefty hardware machine for it. (You can get all the terraform scripts from here.)

_Note: As we only allow IaC to run on Azure, you will be using the terrform scripts provided to provision the resources._ [Help](terraform.md)

## Resource Group
Create a resource group of the format _name-k8s-rg_. This is just for identification and has no implication on how we create our nodes.
```terraform
resource "azurerm_resource_group" "k8s" {
    name     = "gb-k8s"
    location = "westeurope"

    tags = {
        environment = "Lab"
    }
}
```

## Networking
We will be using a single Virtual Network for all our nodes so that the nodes can communicate with each other. If you use different Virtual Networks, that would mean specific network access policies for nodes to communicate between the different Virtual Networks. You will also need a network interface to assign an ip to the nodes.
The ports used by the Control-plane nodes are:

| Protocol | Direction | Port Range | Purpose | Used By |
| --- | --- | --- | --- | --- |
| TCP | Inbound | 6443* | Kubernetes API server | All |
| TCP | Inbound | 2379-2380 | etcd server client API | kube-apiserver, etcd |
| TCP | Inbound | 10250 | Kubelet API | Self, Control plane |
| TCP | Inbound | 10251 | kube-scheduler | Self |
| TCP | Inbound | 10252 | kube-controller-manager | Self |

The ports used by the worker nodes are:

| Protocol | Direction | Port Range | Purpose | Used By |
| --- | --- | --- | --- | --- |
| TCP | Inbound | 10250 | Kubelet API | Self, Control plane |
| TCP | Inbound | 30000-32767 | Nodeport Services | All |

The Nodeport services are nothing but the micro services which we deploy, in k8s terms it's refered to as Nodeport Services.

### Virtual Network
Create a virtual network by logging into the Azure portal or using the terraform script below.

### Subnet
Creating the subnet:
```terraform 
resource "azurerm_subnet" "k8s" {
    name = "gb-k8s-subnet"
    virtual_network_name = "euw-dev-123-dcp-learnedroutes"
    resource_group_name = "euw-dev-123-dcp-net"
    address_prefix = "10.118.20.16/28"
}
```
### Network Security Group
Creating the network security group to be used by the network interfaces:
```terraform
resource "azurerm_network_security_group" "k8s" {
    name="gb-k8s-nsg"
    location = "westeurope"
    resource_group_name="euw-dev-123-dcp-net"
    security_rule {
        access = "Allow"
        direction = "Inbound"
        name = "gb-k8s-inbound"
        priority = 100
        protocol = "TCP"
        source_port_range = "*"
        destination_port_range = "*"
        source_address_prefix = "*"
        destination_address_prefix = "*"
    }
    security_rule {
        access = "Allow"
        direction = "Outbound"
        name = "gb-k8s-outbound"
        priority = 100
        protocol = "TCP"
        source_port_range = "*"
        destination_port_range = "*"
        source_address_prefix = "*"
        destination_address_prefix = "*"
    }

    tags = {
        environment = "Lab"
    }
}
```
### Network Interfaces
Creating the network interfaces for control plane:
```terraform
resource "azurerm_network_interface" "k8s-control-plane" {
    count = 3
    name = "gb-k8s-cp-${count.index}"
    location = "westeurope"
    resource_group_name="euw-dev-123-dcp-net"
    network_security_group_id = "${azurerm_network_security_group.k8s.id}"

    ip_configuration {
        name = "gb-k8s-cp-ip-${count.index}"
        private_ip_address_allocation = "Dynamic"
        subnet_id = "${azurerm_subnet.k8s.id}"
    }

    tags = {
        environment = "Lab"
    }
}
```

Creating the network interfaces for the workers:
```terraform
resource "azurerm_network_interface" "k8s-worker" {
    count = 3
    name = "gb-k8s-w-${count.index}"
    location = "westeurope"
    resource_group_name="euw-dev-123-dcp-net"
    network_security_group_id = "${azurerm_network_security_group.k8s.id}"

    ip_configuration {
        name = "gb-k8s-w-ip-${count.index}"
        private_ip_address_allocation = "Dynamic"
        subnet_id = "${azurerm_subnet.k8s.id}"
    }

    tags = {
        environment = "Lab"
    }
}
```

## Public IP
Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API servers.
```terraform
resource "azurerm_public_ip" "k8s" {
    name                = "gb-k8s-public-ip"
    location            = "westeurope"
    resource_group_name = "${azurerm_resource_group.k8s.name}"
    allocation_method   = "Static"

    tags = {
        environment = "Lab"
    }
}
```

## Creating the Nodes
We will be using three nodes for control plane and three as worker nodes.
Below is the terraform script which creates the nodes for you in the dev subscription. All the nodes will be assigned the same resource group for ease of use.

### Control-Plane Nodes
Creating the nodes for the control-plane:
```terraform
resource "azurerm_virtual_machine" "k8s-control-plane" {
    count = 3
    name = "gb-k8s-cp-${count.index}"
    location = "westeurope"
    resource_group_name="${azurerm_resource_group.k8s.name}"
    network_interface_ids = ["${element(azurerm_network_interface.k8s-control-plane.*.id, count.index)}"]
    vm_size = "Standard_DS1_v2"
    delete_data_disks_on_termination = true
    delete_os_disk_on_termination = true
    storage_image_reference {
        publisher = "Canonical"
        offer = "UbuntuServer"
        sku = "16.04.0-LTS"
        version = "latest"
    }
    storage_os_disk {
        name = "gb-k8s-cp-${count.index}"
        caching = "ReadWrite"
        create_option = "FromImage"
        managed_disk_type = "Premium_LRS"
    }
    os_profile {
        computer_name = "gb-k8s-cp-${count.index}"
        admin_username = "gb"
        admin_password = "1Password"
    }
    os_profile_linux_config {
        disable_password_authentication = false
    }

    tags = {
        environment = "Lab"
    }
}
```
### Worker Nodes
Creating the nodes for the workers:
```terraform
resource "azurerm_virtual_machine" "k8s-worker" {
    count = 3
    name = "gb-k8s-w-${count.index}"
    location = "westeurope"
    resource_group_name="${azurerm_resource_group.k8s.name}"
    network_interface_ids = ["${element(azurerm_network_interface.k8s-worker.*.id, count.index)}"]
    vm_size = "Standard_DS1_v2"
    delete_data_disks_on_termination = true
    delete_os_disk_on_termination = true
    storage_image_reference {
        publisher = "Canonical"
        offer = "UbuntuServer"
        sku = "16.04.0-LTS"
        version = "latest"
    }
    storage_os_disk {
        name = "gb-k8s-w-${count.index}"
        caching = "ReadWrite"
        create_option = "FromImage"
        managed_disk_type = "Premium_LRS"
    }
    os_profile {
        computer_name = "gb-k8s-w-${count.index}"
        admin_username = "gb"
        admin_password = "1Password"
    }
    os_profile_linux_config {
        disable_password_authentication = false
    }

    tags = {
        environment = "Lab"
    }
}
```

Ideally you should use key based ssh access but for this exercise, it's easier to create the nodes with username and password.

Once you are done, ssh into the instances just as a sanity check and keep the ip addresses of the nodes at hand for reference.

# Verify
SSH into one of the nodes and ping to the other VMs with their hostnames. You should be able to get response from the remote machines.

### Connect to the jumpbox
```bash
ssh vm-user@10.118.20.4
```

### Finally _tmux_
```tmux``` is a terminal multiplexer for UNIX based operating systems. With one _ssh_ connection you will be able to open multiple windows doing multiple things.

To create a new ```tmux``` session:
```bash
tmux new -s <session-name>
```

To list all ```tmux``` sessions:
```bash
tmux ls
```

To detach from ```tmux``` session:
```bash
(Ctrl + b) + d
```

To split ```tmux``` _panes_ horizontally:
```bash
(Ctrl + b) + “
```

To split ```tmux``` _panes_ vertically:
```bash
(Ctrl + b) + %
```

To resize a ```tmux``` _pane_:
```bash
(Ctrl + b) + :
```
>Then enter ```resize-pane [direction] [length]```. Sample resize commands would include:
```bash
:resize-pane -D (Resizes the current pane down)
:resize-pane -U (Resizes the current pane upward)
:resize-pane -L (Resizes the current pane left)
:resize-pane -R (Resizes the current pane right)
:resize-pane -D 10 (Resizes the current pane down by 10 cells)
:resize-pane -U 10 (Resizes the current pane upward by 10 cells)
:resize-pane -L 10 (Resizes the current pane left by 10 cells)
:resize-pane -R 10 (Resizes the current pane right by 10 cells)
```
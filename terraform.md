# Terraform hack sheet

## File structure
To run a terraform script, create a folder and create the _main.tf_, _provider.tf_, and _variables.tf_ files inside it.
Sample file structure:
![terraform folder structure](https://github.dev.global.tesco.org/raw/IN22912599/k8s-lab/master/images/folder-structure-terraform.png)

## To specify a provider
``` terraform
provider "azurerm" {
    subscription_id = "${var.subscription_id}"
    tenant_id       = "${var.tenant_id}"
    client_id       = "${var.client_id}"
    client_secret   = "${var.client_secret}"
}
```

## Check your script with Terraform
Before running the terraform script on the cloud provider, it is always advisable to ```plan``` the terraform script to do a sanity test of the script. Terraform has an sequential interpreter and thus does not compile the entire file at once, hence errors at the later parts of the terraform file will not be identified unless the previous resources has been created. To work around this problem of corrupting the cloud provider with incorrect resources, terraform saves the state of the provision every time a resource is created successfully. So, after encountering an error, when terraform is resumed, the execution starts from the saved state.

```
cd <to-terraform-directory>
terraform plan
```

>Note: There has been instances where terraform still screws up the resource provisioning, so it is always advisable to run the plan first

## Provision resources with Terraform
Use the following command to create all your resources:
```
cd <to-terraform-directory>
terraform apply
```

## Destroy resources with Terraform
Use the following command to destroy all resources created earlier:
```
cd <to-terraform-directory>
terraform destroy
```


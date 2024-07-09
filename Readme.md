Video Courtesy: https://www.youtube.com/watch?v=lH3KT9RUEOA&list=PLLc2nQDXYMHowSZ4Lkq2jnZ0gsJL3ArAw&index=1

The first 4 videos are just Introduction of Azure and terraform so I had nothing much to share and the actual content starts from the 5th.

### Video: 5 - Creating a Resource group
The reason behind creating the app registration in Azure is “This registration **provides your application with the necessary credentials to securely access Azure services and APIs, making your applications and services more secure and accessible**“
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/cf232cbf-dd1e-4999-a45b-e351d3e3a150)
Select the App registration.
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/590691d4-1dab-421d-8bcb-03862e46fad0)
Give a name for the app and select register.
We are creating an identity in Entra ID which can be used in the terraform configuration file and Azure can authenticate the same.
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/b7f5a019-8c00-4d13-900a-d4a0ddb60121)
We can get the client id and the tenant id from this tab and we can create the client secret with the client credentials.
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/0e27e4a2-6cbf-4afd-bba1-9370915f3440)
Generate the secret and add it the terraform configuration.
```json
//main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.109.0"
    }
  }
}
provider "azurerm" {
  subscription_id = ""
  client_id = ""
  client_secret = ""
  tenant_id = ""
  features {}
}

resource "azurerm_resource_group" "app_grp"{
  name="app-grp"
  location="canadacentral"
}
```

![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/8011504e-9114-4e20-afb3-9c08f767b83c)
Now Azure throws an error stating that the authentication has failed. This has happened because we need a role based access control that needs to be assigned to the application object “terraform”.  This will let give terraform(application object) to access the resources in Azure.
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/00b0f8f1-a15c-4284-bcaf-005cda2cd924)
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/249e5f7c-de25-4062-9823-090a99aa4074)
In the role assignment add the member terraform → click on Select and you can see the members are added.
![image](https://github.com/karthi770/Azure_VNet_NSG_Bastion/assets/102706119/1fa21fb1-67f8-489e-aaa3-11df7f93c185)
Click on Review + assign.
Now run the command terraform plan and apply that will create the resource group.

### <u>Video: 6 - Storage Account </u>

The default parameters for creating the storage account are:
- Resource group
- Location
- Name
- Performance
- Redundancy
```json
//main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.109.0"
    }
  }
}

provider "azurerm" {
  subscription_id = ""
  client_id = ""
  client_secret = ""
  tenant_id = ""
  features {}
}

resource "azurerm_resource_group" "app_grp"{
  name="app-grp"
  location="canadacentral"
}

resource "azurerm_storage_account" "az-storage" {
  name                     = "terraformstorage1092"
  resource_group_name      = azurerm_resource_group.app_grp.name
  location                 = azurerm_resource_group.app_grp.location
  account_tier             = "Standard"
  account_replication_type = "LRS" //Locally redundant storage
  tags = {
    environment = "staging"
  }
}
```

There are different configuration settings in storage account and they can be passed as arguments in storage account resource.

The default value of Allow Blob public Enabled is false

### <u>Video: 7 - Terraform state</u>
We saw in the previous video that the default value of Allow Blob public Enabled is false.

Now if we manually change the value to true in the Azure portal then it is change of Terraform state which will be reflected when we try to other operations in Terraform.

Understand more about state locking in Azure from the below link:
https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage?tabs=azure-cli

### <u>Video: 8 - Creating a container and Blob:</u>

```json
resource "azurerm_storage_container" "data_container" {
  name                  = "data"
  storage_account_name  = azurerm_storage_account.az-storage.name
  container_access_type = "private"
}

resource "azurerm_storage_blob" "az_blob" {
  name                   = "my-awesome-content.zip"
  storage_account_name   = azurerm_storage_account.az-storage.name
  storage_container_name = azurerm_storage_container.data_container.name
  type                   = "Block"
  source                 = "some-local-file.zip"
```
The container access type is the level of access to the object that is uploaded in the container.  Now the option is Private which means we have no access to the object, once the container_access_type is changed to “blob” we can download the object from the URL that has been assigned to the object.
### <u>Video: 9 - Dependencies between</u>
In every resource block we need to specify the depends on parameter so that when terraform creates the resources it makes sure that the dependencies are met while creating the resources. For example: if we need to create a blob storage we need to make sure that we have a container to store the blob. That can be specified in the block of code shown below:
```json
  name                   = "my-awesome-content.zip"
  storage_account_name   = azurerm_storage_account.az-storage.name
  storage_container_name = azurerm_storage_container.data_container.name
  type                   = "Block"
  source                 = "some-local-file.zip"
  depends_on = [ azurerm_storage_container.data_container ]
}
```

### <u>Video: 10 - Deleting the resources</u>
`terraform destroy` command will delete the resource group which means this will delete all the resources associated with it, so we need to be mindful before using this command.

### <u>Video: 11 - Using Variables</u>
```json
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.109.0"
    }
  }
}

provider "azurerm" {
  subscription_id = ""
  client_id = ""
  client_secret = ""
  tenant_id = ""
  features {}
}

resource "azurerm_resource_group" "app_grp"{
  name=local.resource_group_name
  location=local.location
}

resource "azurerm_storage_account" "az-storage" {
  name                     = var.storage_account_name //"terraformstorage1092"
  resource_group_name      = local.resource_group_name
  location                 = local.location
  account_tier             = "Standard"
  account_replication_type = "LRS" //Locally redundant storage
  public_network_access_enabled = true
  depends_on = [ azurerm_resource_group.app_grp ]
}

resource "azurerm_storage_container" "data_container" {
  name                  = "data"
  storage_account_name  = var.storage_account_name
  container_access_type = "private"
  depends_on = [ var.storage_account_name ]
}

resource "azurerm_storage_blob" "az_blob" {
  name                   = "my-awesome-content.zip"
  storage_account_name   = var.storage_account_name
  storage_container_name = azurerm_storage_container.data_container.name
  type                   = "Block"
  source                 = "some-local-file.zip"
  depends_on = [ azurerm_storage_container.data_container ]
}

variable "storage_account_name" {
  type = string
  description = "Name of the storage account"
}

locals {
  resource_group_name = "app-grp"
  location = "canadacentral"
}
```
The changes that have been made in the above code are, we have added block called variables, which carries a string and when we run the `terraform plan` command this will prompt us to enter the string (in our case it is storage account name).

We have also added a depends on argument inside the code block and mention the storage account can be created only after the resource group is created.

We have added a locals block where we can add variables that can be used in the current terraform configuration file. Ex: We have declared the resource group name and location in the locals and used it in other block of code.






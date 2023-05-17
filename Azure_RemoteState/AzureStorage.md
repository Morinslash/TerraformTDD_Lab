# Using Azure storage for Terraform Remote State

## Azure blob storage
- Remote location of state file on Azure Cloud
- Supports locking file
- Supports workspaces with Terraform
- Multiple Authentication Methods

### Authentication Methods for accessing state file

| Method                   | Configuration block           |
|--------------------------|-------------------------------|
| Managed Service Identity | `use_msi`                     |
| Shared Access Signature  | `sas_token`                   |
| Storage Access Key       | `access_key`                  |
| Service Principal        | `client_id` + `client_secret` |

### Remote State Storage Setup

The [main.tf](./main.tf) file contains Terraform script that will create for us storage account in which we can hold our terraform state. It will also generate for us SAS Token for authentication and permissions to read, write the state files.

#### Create Resource Group for 

First we will have to create the Resource Group to create our Storage Account

```terraform
resource "azurerm_resource_group" "setup" {
  name     = var.resource_group_name
  location = var.location
}
```

#### Create Storage Account

Storage Account will be created based on the variable `naming_prefix` variable and random number in range 10000 - 99999 to provide globally unique name for it.

```terraform
resource "azurerm_storage_account" "terraform_storage_account" {
  name                     = "${lower(var.naming_prefix)}${random_integer.sa_num.result}"
  resource_group_name      = azurerm_resource_group.setup.name
  location                 = var.location
  account_tier             = "Standard" //We don't need more
  account_replication_type = "LRS" // replication depends on needs
}
```

#### Create Container for JSON

Now we need to create Container in which we will store our state file

```terraform
resource "azurerm_storage_container" "state_container" {
  name                 = "terraform-state"
  storage_account_name = azurerm_storage_account.terraform_storage_account.name
}
```

#### Allocate SAS Token

Lastly we will create SAS Token to enable access from other scripts to the state file

```terraform
data "azurerm_storage_account_sas" "state" {
  connection_string = azurerm_storage_account.terraform_storage_account.primary_connection_string
  https_only        = true

  resource_types {
    service   = true
    container = true
    object    = true
  }

  services {
    blob  = true
    queue = false
    table = false
    file  = false
  }

  start  = timestamp()
  expiry = timeadd(timestamp(), "17520h")

  permissions {
    read    = true
    write   = true
    delete  = true
    list    = true
    add     = true
    create  = true
    update  = false
    process = false
  }
}
```

*note that this is `not a resource` type but `data` type*

#### Outputs and Sharing access

As the last step we have to get information about SAS Token and location of state file so we can use it in other scripts. We will generate file for it

```terraform
resource "local_file" "post-config" {
  depends_on = [azurerm_storage_container.state_container]

  filename = "${path.module}/backend-config.txt"
  content  = <<EOF
storage_account_name = "${azurerm_storage_account.terraform_storage_account.name}"
container_name = "terraform-state"
key = "terraform.tfstate"
sas_token = "${data.azurerm_storage_account_sas.state.sas}"

  EOF
}
```

```terraform
output "storage_account_name" {
  value = azurerm_storage_account.terraform_storage_account.name
}

output "resource_group_name" {
  value = azurerm_resource_group.setup.name
}
```
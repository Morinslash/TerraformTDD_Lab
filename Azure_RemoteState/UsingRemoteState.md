# How to use remote state

## Configuration block

We should use the values from generate file with backend-config.txt, according to instruction from [AzureStorage](./AzureStorage.md)

```terraform
terraform {
  backend "azurerm" {
    storage_account_name = "example_storage_account"
    container_name = "terraform-container"
    key = "terraform.tfstate"
  }
}
```

_**Note: We cannot use variables in configuration backend! This section is evaluated before Terraform initialize variables!**_

### Setting configuration values dynamically:
```bash
terraform init -backend-config="sas_token=example_sas_token_value"
```

### Using the backend-config.txt

```bash
terraform init -backend-config=backend-config.txt
```
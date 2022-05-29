---
title: Using Table Storage as an Alternative to Remote State
layout: post
tags:
  - Azure
  - Azure Table Storage
  - Terraform
---

Terraform is a fantastic tool for Infrastructure as Code.
From the YAML-like HCL syntax (no JSON!), to importing files (linting JSON files FTW!), to retrieving the results of previous runs to link resources, Terraform has made a massive difference in my work.
However, like all technologies, it is not without its weaknesses.
Terraform uses state files to keep track of what the world looked like when it last ran, which is wonderful for identifying drift.
The default pattern is to use these state files for passing data between Terraform modules.
But this is actually an anti-pattern, for HashiCorp [recommend _not_ using remote state for passing data][no-remote-state], in large part because to read the outputs from a state file the caller must have full access to read the entire remote state file, which include secrets they probably shouldn't be allowed to access.

So what to do in Azure?
HashiCorp recommend using SSM Parameter Store for AWS, but for Azure they suggest using resource-specific data sources and provide DNS as an example.
(And yes, I am aware that one could [use Azure DNS as a database][route53-as-db] to store outputs, but I do not advise this.)

Enter one of the least known/used Azure services (maybe?): [Table Storage][table-storage].
I'm sure that part of the reason for its lack of use is that Microsoft is pushing people towards Cosmos DB and the Table API, but Table Storage still exists in v2 storage accounts.

Azure Table storage stores "structures NoSQL data" as key/value pairs, and does so in an existing storage account.
Which means you can use the same storage account to save your state files and the outputs needed by other modules.
The AzureRM provider already includes all the configuration we need to make use of Table storage also: both the `azurerm_storage_table_entity` [resource][table-res] and [data source][table-data] exist.
So how do we go about using it?

Simply, instead of defining `output` blocks in your Terraform code, instead define an `azurerm_storage_table_entity` resource.
In the resource we provide the name of our storage account, and the name of a Table into which we will store rows.
Every table storage row requires a unique combination of `partition_key` and `row_key`.
(While a high-performance application would need to be careful to balance rows evenly with the partition key, reading state outputs does not have this same requirement).

So what's the best way to export the state?
This is how I do it.

```hcl
resource "azurerm_storage_table_entity" "resource" {
  storage_account_name = "myterraformstatestore"
  table_name           = "this_module_name"
  partition_key        = "resource_name"
  row_key              = "region_or_instance"

  entity = {
    id                  = my_resource.id
    name                = my_resource.name
    resource_group_name = my_resource.resource_group_name
    useful_attribute_1  = my_resource.useful_attribute_1
    useful_attribute_2  = my_resource.useful_attribute_2
  }
}
```

So what's going on here?

First, we are using the same storage account for our Table storage as we use to store our state files.
Why?
Because it becomes one less storage account to create, manage, and remember.
Since Table storage is separate from the blobs which store our state files, we can apply different permissions to the Tables and the Blobs (or at least we will be able to when Terraform supports RBAC authentication to the storage account).

Second, we have created a table for our current module.
This can be created as a resource inside the module, meaning its existence can be guaranteed as part of the deployment.
(If you are reusing the same module code for multiple environments, we need to make the table name instance-specific.)

Then we create a row in the table for each instance of our resource, in the same way we would create outputs for each resource.
By using the partition key to refer to the resource name, we can use the row key to identify different instances, for example in a `for_each` or `count` block.

Finally, we build our `entity` which contains all the properties we might want to use in from this resource.
There are three properties I always include:

- `id` as this is the most frequently used property in other resources.
- `name` and `resource_group_name` as these are the typical properties required to fetch all attributes in a `data` resource. (If a resource requires extra properties to allow us to use the data resource, I will include them also.)

After this, we can include any other properties that we expect to frequently use.
For example, we might save the `address_prefixes` from virtual network or subnet for use in firewall rules.

Having stored our attributes in Table storage, we need to use the data resource to retrieve them in another module.

```hcl
data "azurerm_storage_table_entity" "resource" {
  storage_account_name = "myterraformstatestore"
  table_name           = "source_module_name"
  partition_key        = "resource_name"
  row_key              = "region_or_instance"
}
```

Having loaded our outputs from storage, we can access the properties with standard naming: `data.azurerm_storage_table_entity.resource.entity.id` and so forth.

So there we have it.
As an added benefit, I've found that reading from Table storage is faster than reading from remote state, and using Table storage opens up a whole range of other possible scenarios as it can be integrated easily with other systems (unlike a remote state file).

[no-remote-state]: https://www.terraform.io/language/state/remote-state-data#alternative-ways-to-share-data-between-configurations
[route53-as-db]: https://www.lastweekinaws.com/blog/route-53-amazons-premier-database/
[table-storage]: https://docs.microsoft.com/en-us/azure/storage/tables/table-storage-overview
[table-res]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_table_entity
[table-data]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/storage_table_entity

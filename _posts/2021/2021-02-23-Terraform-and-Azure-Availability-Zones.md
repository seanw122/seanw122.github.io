---
layout: posts
title:  "Terraform + Azure Availability Zones"
date:   2021-02-23 15:09:00 +0000
categories: 
  - Terraform
  - Azure
  - Availability Zones
---

While learning Terraform some time back, I wanted to leverage Availability Zones in Azure. I was specifically looking at Virtual Machine Scale Sets. [Terraform Azure Virtual Machine Scale Sets](https://www.terraform.io/docs/providers/azurerm/r/virtual_machine_scale_set.html) Looking at the documentation Terraform has, I noticed there is no good example on using zones. So, I tried a few things to see what was really needed for that field. While doing some research, I noticed there are many people in the same situation. No good examples. I figured I'd create this post to help anyone else. And, of course, it's a good reminder for me too if I forget the syntax on how I did this.

## Body

While learning Terraform some time back, I wanted to leverage Availability Zones in Azure. I was specifically looking at Virtual Machine Scale Sets. [Terraform Azure Virtual Machine Scale Sets](https://www.terraform.io/docs/providers/azurerm/r/virtual_machine_scale_set.html) Looking at the documentation Terraform has, I noticed there is no good example on using zones. So, I tried a few things to see what was really needed for that field. While doing some research, I noticed there are many people in the same situation. No good examples. I figured I'd create this post to help anyone else. And, of course, it's a good reminder for me too if I forget the syntax on how I did this.

Here's a very simple Terraform file. I just created a new folder then a new file called zones.tf. Here's the contents:

``` terraform
variable "location" {
    description = "The location where resources will be created"
    default = "centralus"
    type = string
}

locals {
    regions_with_availability_zones = ["centralus","eastus2","eastus","westus"]
    zones = contains(local.regions_with_availability_zones, var.location) ? list("1","2","3") : null
}

output "zones" {
    value = local.zones
}
```

The variable `location` is allowed to be changed from outside the script. But, I used `locals` for variables I didn't want to be changed from outside. I hard coded a list of Azure regions that have availability zones. Right now it's just a list of regions in the United States. Of course, this is easily modifiable to add other regions.

The `zones` local variable uses the contains function to see if the specified region is in that array. If so, then the value is a list of strings. Else it's null. This is important. The zones field in Azure resources required either a list of strings or null. An empty list didn't work for me.

As it is right now, you can run the Terraform Apply command and you should see some output. Changing the value of the location variable to something not in the list and you may not see output at all simply because the value is null.

Now, looking at a partial example from the Terraform documentation:

``` terraform
resource "azurerm_virtual_machine_scale_set" "example" { 
    name = "mytestscaleset-1" 
    location = var.location
    resource_group_name = "${azurerm_resource_group.example.name}" 
    upgrade_policy_mode = "Manual" 
    zones = local.zones
}
```

Now the zones field can be used safely when the value is either a list of strings or null. After I ran the complete Terraform script for VM Scale Set, I went to the Azure Portal to verify it worked.

This proved to me that I can use a region with and without availability zones in the same Terraform script.

For a list of Azure regions with Availability Zones, see: [Azure Availability Zones](https://docs.microsoft.com/en-us/azure/availability-zones/az-overview){:target="_blank"}

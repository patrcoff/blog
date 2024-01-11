+++
title = 'Exploring_azure_logic_apps'
date = 2024-01-04T21:20:26Z
draft = true
+++

# Exploring Azure Logic Apps (Standard)

## Background, Deploying with Terraform and Working with Private Networks (Part 1)

In this post, I shall discuss what I have learned and what I'm still struggling with regarding Logic Apps, which I've been exploring as part of a project at work.

For context, [(skip if you don't care)](#what's-in-this-post?) I have recently transitioned from SysAdmin/IT Support to Platform Engineering and this is one of the first things I have looked at in a real world project for a client.

There is much I am yet to learn, not just in terms of Logic Apps themselves but also more broadly in Azure, Terraform and DevOps culture/practices in general, so this isn't designed to be straight up technical 'How-To' or the words of a leading expert. That said I endeavour to be as clear as possible about what I do know for sure and what is speculation on my part/based on certain assumptions which may or may not be correct. I also include my own thoughts and opinions  about my experience working with Logic Apps (mainly from an infrastructure point of view) in the hope that it is useful or interesting.

### What's in This Post?

This particular post will cover an overview of Logic Apps, the deployment of the required infrastructure via Terraform and some basic Workflow creation using the portal UI designer. I intend to write further posts in the series covering additional related topics such as using the VSCode extension and delving deeper into some of the different connectors available. When they are created I shall link them here.

One last piece of important context before we begin proper; as part of the client's requirements, I have had to explore deploying a Logic App Standard into a **private Azure virtual network** where public endpoints are completely banned for all resources, including for the Storage Account which stores the Logic Apps state and workflows (don't worry, I'll cover all this later). If your needs are less restrictive then simpler infrastructure and even maybe Logic Apps Consumption would suffice. I shall revisit the infrastructure deployment with relation to more relaxed requirements in a future piece. For now I'll link to [Microsoft's docs](https://learn.microsoft.com/en-us/azure/logic-apps/single-tenant-overview-compare) on the differences between Standard and Consumption for those who are interested.


## What are Logic Apps anyway?

Logic Apps are an Azure Service which provides a no-code/low-code solution for integrating numerous different services and resources. Basically, it's a platform for gluing together disparate APIs from a wide range of Azure and 3rd party services using a visual workflow UI.



> [!INFO] In Microsoft's Words
> Azure Logic Apps is a cloud platform where you can create and run automated workflows with little to no code. By using the visual designer and selecting from prebuilt operations, you can quickly build a workflow that integrates and manages your apps, data, services, and systems.


For example, you could create a Workflow in a Logic App which polls an email provider for new emails, runs sentiment analysis on the content using Azure's ML offerings, posts results to a message queue and sends a notification to a key stakeholder if the content of the message was deemed negative. As you can see, pulling together a bunch of different services in this (intended-to-be) easy way can be rather powerful, allowing your customer relations manager to keep on top of complaints or automating your Finance Manager's strange and archaic 'system' for managing invoices and, I don't know, counting them up or whatever Finance Manager's do... Joking aside, the PAAS solution is very powerful and flexible, if not a little quirky at times, which we'll get into throughout.

It would be prudent of me to mention that my half baked example scenarios are really just the tip of the ice-berg and the solution is designed for a huge scope of use cases, from very legacy B2B type integrations to much more lightweight automatons and process improvements. It won't be possible to cover nearly as many possible use cases as I'd like (I don't have the time to fully understand the half of them) and how you implement your Logic Apps infrastructure can be influenced by your use case, so my examples won't fit all sizes but hopefully will give you a decent idea of what is possible, and how to do it.

## [What About Microsoft's other offerings?](https://learn.microsoft.com/en-us/azure/azure-functions/functions-compare-logic-apps-ms-flow-webjobs)

You might have heard of several other options for automation and service integration from Microsoft and it can be a little confusing knowing which to go for. For example, at first glance, Logic Apps sounds potentially quite similar to Power Apps or Power Automate. An overly simplistic way to think about it would be that the Power platform is intended more towards IT admins who have O365 in their organisations or even technical minded office workers from other departments. Logic Apps is obviously targeted at those with full Azure subscriptions, and generally more 'cloud developer' type roles, although still aimed at the low-code angle. (By 'cloud developer' I am being _very_ loose, this would mean anyone from software engineers to people like myself, Platform Engineers, and all sorts of DevOps-y roles in between).

Functions apps is another Azure service you might have heard of, Logic Apps is actually built on top of Azure Functions in its internal workings though you aren't required to understand Functions in order to use them. Both Logic Apps and Azure Functions are Azure platforms for running **_serverless_** workloads but Functions is defined as a serverless _compute_ service rather than a serverless _workflow integration platform_, as Logic Apps is. Basically, Functions is going to be a lot less low-code (or, more-code you might say) and is geared more towards the developer end of the spectrum than the IT-Pro end, and is more generalised whereas Logic Apps is more specialised in its use cases.


## Configurations

Well, assuming you're still with me, we've settled on Logic Apps and are going to deploy one right away? Hold your horses! There are a _few_ choices we have to make first.

> Options, Oh the Options...

Logic Apps, while I have just called them '_specialised_', are still pretty massive in terms of configuration and deployment options. Firstly, you have to choose between **_Consumption_** and **_Standard_** Logic Apps. What the hell does that even mean I hear you ask (well at least, I did when I first looked into them).

Disregarding the ridiculous naming, these terms split Logic Apps into two very different categories in terms of **_hosting_** - Consumption or _multi-tenant_ Logic Apps (I'm sure that's _helpful_) basically run as a fully managed service in the public cloud. Crucially, they run on _shared compute_ resources in the public cloud. They are the cheapest option and you basically just define your workflow (a single workflow) per Logic App Consumption resource. Hence, they are not designed for high levels of usage as their performance is not as guaranteed as Logic Apps Standard and has much lower limits. This is all based on the theory I should say (which does make sense) as I have not stress tested any from either resource version.

### Logic Apps Standard Explained

Logic Apps Standard (I do not like that naming convention but who cares what I think) are run in **_single_-tenant** meaning on compute resources within your own **subscription** which you have more control over. Within them, there are still further decisions to be made however. You can deploy Logic Apps either into specific App Service Plan skus, an App Service Environment or within a container on a Linux system*


> [!IMPORTANT]
> *Containerised Logic Apps are intended to be run within Azure Arc for production. While you can use the containerised runtime for local development, it is **not supported**  outside of a proper Kubernetes cluster with the ARC service enabled. It should also be noted that this option is currently still in Preview and so is subject to potential future breaking changes.


Using App Service Plan is sort of the middle ground (between consumption and other Standard tier hosting options) in terms of level of control over compute/networking isolation, performance and cost and is likely what most enterprises would need, at least initially. Of course, if you already heavily make use of Kubernetes, it might make more sense to use the containerised runtime but please note the above caveats.

With ASP, you can select a specialised tier of 'WS1', 'WS2' or 'WS3' for different compute sizes to host your Logic App. You install one Logic App Standard into an ASP, however, you can create as many Workflows as you like within it up to the limit of the compute/networking specs deployed to (off the top of my head this is 50). Crucially, ASP hosted Logic Apps can make use of **Private Endpoints** for ingress traffic and **VNET integration** for egress traffic allowing integration with private VNETs and even on-prem services (via existing network infrastructure such as site-to-site VPN). This is a crucial difference from Consumption tier Logic Apps (while some level of isolation and private network integration is possible for Consumption Logic Apps via 'Integration Service Environments', this is deprecating soon and so won't be discussed and I don't advise you waste any time on it).

> [!INFO]
> Another option you have within Standard tier is App Service Environment (ASE). This is basically a beefier orchastrator of ASPs and comes with a heck of a jump in running costs and complexity so I won't go into it either. ASEs are generally needed when significant numbers of ASP instances are requires (>30) to host your runtimes or when very strict controls exist around data compliance as they are fully isolated in terms of physical compute and networking from any other cloud users.



## Finally, Deployment!

Ok, I think we've suffered enough definitions and explanations for now, let's get to actually deploying a Logic App using Terraform. Here I shall outline the deployment of the basic dependencies of the Logic App, starting with a module containing the Logic App itself.

```#logic_app_module/versions.tf
terraform {
  # - need to confirm min version
  required_version = ">= 0.12"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">=3.84.0"
    }
  }
}
```

```#logic_app_module/logic_app.tf

resource "azurerm_service_plan" "app" {
  name                = "${var.name}-service-plan"
  resource_group_name = data.azurerm_resource_group.rg.name
  location            = data.azurerm_resource_group.rg.location
  os_type             = var.app_service_os_type # Must be Windows if not using containerised Logic App runtime
  sku_name            = var.app_service_plan_tier
}

resource "azurerm_logic_app_standard" "app" {
  name                       = var.name
  location                   = var.location
  resource_group_name        = var.resource_group_name
  app_service_plan_id        = azurerm_service_plan.app.name
  storage_account_name       = var.storage_account_name
  storage_account_share_name = var.storage_account_share_name
  storage_account_access_key = var.primary_access_key
  https_only                 = true
  virtual_network_subnet_id  = var.subnet_id
  version                    = "~4"

  identity {
    type = "SystemAssigned"
  }


  app_settings = {
    "FUNCTIONS_WORKER_RUNTIME" = "node"
    "WEBSITE_NODE_DEFAULT_VERSION" = "~18"
    "WEBSITE_DNS_SERVER"           = var.WEBSITE_DNS_SERVER

  }
  site_config {
    vnet_route_all_enabled = var.WEBSITE_VNET_ROUTE_ALL
  }

  #depends_on = [ azurerm_service_plan.app ]

}

data "azurerm_resource_group" "rg" {
  name = var.ResourceGroupName
}

```

```#logic_app_module/private_endpoint.tf

resource "azurerm_private_endpoint" "main" {
  name                = "${var.name}-pe"
  resource_group_name = data.azurerm_resource_group.rg.name
  location            = data.azurerm_resource_group.rg.location
  subnet_id           = var.pep_subnet_id
  tags                = var.tags

  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [var.private_dns_zone_id]
  }

  private_service_connection {
    name                           = "private-connection-${var.name}"
    private_connection_resource_id = azurerm_logic_app_standard.app.id
    subresource_names              = ["sites"]
    is_manual_connection           = false
  }

  depends_on = [
    azurerm_logic_app_standard.app,
  ]
}
```


```#logic_app_module/variables.tf

variable "name" {
  type        = string
  description = "Logic App Name."
}

variable "location" {
  type    = string
  default = "northeurope"
}

variable "storage_account_name" {
  type        = string
  description = "The storageAccountName the Logic App is using."
}

variable "storage_account_share_name" {
  type        = string
  description = "The name of the storage account share the Logic App is using"
}

variable "resource_group_name" {
  type = string
}

variable "tags" {
  type        = map(string)
  description = "Map of Tags"
  default     = {}
}


variable "WEBSITE_DNS_SERVER" {
  description = "DNS server app setting"
  type        = string
  default     = "10.245.36.89"
}

variable "subnet_id" {
  description = "Subnet resource id for vnet integration."
  type        = string
  default     = ""
}

variable "private_dns_zone_id" {
  description = "ID of the private DNS zone"
  type        = string
}

variable "pep_subnet_id" {
  description = "Subnet ID for the private end point"
  type        = string
}

variable "app_service_os_type" {
  type    = string
  default = "Windows"
  validation {
    condition     = contains(["Windows", "Linux"], var.app_service_os_type)
    error_message = "app_service_os_type must be one of 'Windows' or 'Linux'"
  }
  description = "OS type for the app service host - must be Windows unless using containerised logic apps runtime (not yet implemented)"
}

variable "app_service_plan_tier" {
  type        = string
  default     = "WS1"
  description = "Sku for app service plan i.e. WS1/WS2/WS3"

  validation {
    condition     = contains(["WS1", "WS2", "WS3"], var.app_service_plan_tier)
    error_message = "app_servicce_plan_tier must be one of 'WS1', 'WS2' or 'WS3'. "
  }
}
```

```#logic_app_module/outputs.tf

output "logicapp_resourceid" {
  value       = azurerm_logic_app_standard.app.id
  description = "The Logic App Resource ID. Needed for Private Endpoints"
}

output "private_endpoint_ip" {
  description = "Private IP of the logic app's private endpoint, required for dns record"
  value       = azurerm_private_endpoint.main.private_service_connection.0.private_ip_address
}

output "logic_app_outbound_ips" {
  description = "List of outbound IPs from the Logic App for use in whitelisting in NSGs and Firewalls."
  value       = azurerm_logic_app_standard.app.outbound_ip_addresses
}
```




#### Demo Environment Deployment

```#main.tf

# We strongly recommend using the required_providers block to set the
# Azure Provider source and version being used
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.84.0"
    }
  }
}

# Configure the Microsoft Azure Provider
provider "azurerm" {
  #skip_provider_registration = true # This is only required when the User, Service Principal, or Identity running Terraform lacks the permissions to register Azure Resource Providers.
  features {}
}

# provider "http" {

# }

resource "azurerm_resource_group" "rg" {
  name     = "${local.appname}-rg"
  location = var.location
}
```

```#network.tf
resource "azurerm_virtual_network" "vnet" {
  resource_group_name = azurerm_resource_group.rg.name
  name                = "${local.appname}-vnet"
  address_space       = ["10.0.0.0/16"]
  dns_servers         = ["168.63.129.16"]
  location            = azurerm_resource_group.rg.location
}

# Used for bastions, jumpboxes etc to test basic internal comms with service being tested
resource "azurerm_subnet" "developer_subnet" {
  name                 = "${local.appname}-dev-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.15.0/24"]
}

resource "azurerm_subnet" "private_endpoints" {
  name                 = "${local.appname}-pe-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.10.0/24"]

  private_endpoint_network_policies_enabled     = true  #??
  private_link_service_network_policies_enabled = false #??
}

resource "azurerm_subnet" "logic_app_subnet" {
  name                                          = "${local.appname}-la-subnet"
  resource_group_name                           = azurerm_resource_group.rg.name
  virtual_network_name                          = azurerm_virtual_network.vnet.name
  address_prefixes                              = ["10.0.5.0/24"]
  private_endpoint_network_policies_enabled     = false #??
  private_link_service_network_policies_enabled = false #??

  delegation {
    name = "serverFarms"

    service_delegation {
      name    = "Microsoft.Web/serverFarms"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}
```

```#dns.tf

# DNS zones and links for thee storage account services and logic app
resource "azurerm_private_dns_zone" "private_dns_zone" {
  for_each            = local.dns_zones
  name                = each.value.name
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "vnet_link" {
  for_each              = local.dns_zones
  name                  = "${local.appname}-vnet-${each.key}-link"
  resource_group_name   = azurerm_resource_group.rg.name
  private_dns_zone_name = azurerm_private_dns_zone.private_dns_zone[each.key].name
  virtual_network_id    = azurerm_virtual_network.vnet.id
}
```

```#storage_account.tf

# for logic app state
resource "azurerm_storage_account" "logic_app_state" {
  name                     = local.storage
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS" #remember, this is a testing environment, not prod!
  min_tls_version          = "TLS1_2"
  account_kind             = "StorageV2"

  allow_nested_items_to_be_public = false
  public_network_access_enabled   = false

  network_rules {
    default_action = "Deny"
    ip_rules       = []
    bypass         = ["AzureServices"]
  }

  # needed when deploying from my laptop due to a tf known issue deploying shares to private storage accounts
  provisioner "local-exec" {
    command = <<EOF
az storage share-rm create \
    --resource-group ${azurerm_resource_group.rg.name} \
    --storage-account ${local.storage} \
    --name ${local.storage}-content \
    --quota 1024 \
    --enabled-protocols SMB \
    --output none  
EOF
  }

}
```

In the above file, we create first the storage account itself, but once it is created we also create a fileshare using the local-exec provisioner. This might seem a little strange but the problem is as follows:

- We are creating a Storage Account which has no public access (only via the private endpoints we'll see shortly)
- Our computer is not connected directly to the VNET we've created.
- Terraform's fileshare resource unfortunately attempts to make a direct connection to the fileshare service within the Storage Account to confirm if a share with that name already exists or not, before attempting to create it. This is problematic as this API call goes over the public internet when we're deploying from a network which isn't connected to the VNET. 

Thankfully the az cli includes a command to create a fileshare via **resource manager** (that's a topic for another time) which does not attempt to reach the share service over its endpoint. This is what terraform usually does to create resources behind the scenes, just not in this specific case. I am not sure if this is actually a bug or not, but for now, our little hack should work ok.




```private_endpoints.tf


resource "azurerm_private_endpoint" "storage_private_endpoints" {
  for_each            = toset(local.storage_subresources)
  name                = "${azurerm_storage_account.logic_app_state.name}-${each.key}-pe"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "${azurerm_storage_account.logic_app_state.name}-${each.key}-psc"
    private_connection_resource_id = azurerm_storage_account.logic_app_state.id
    is_manual_connection           = false
    subresource_names              = [each.key]
  }

  private_dns_zone_group {
    name                 = "${local.appname}-${each.key}-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.private_dns_zone[each.key].id]
  }
}
```


```#call_module.tf

module "logic_app" {
  source = "../pos-3202/modules/logic_app_module_cle"

  name     = "${local.appname}-la"
  location = var.location
  storageAccountName      = local.storage
  storageAccountShareName = "${local.storage}-content"
  pep_subnet_id           = azurerm_subnet.private_endpoints.id
  private_dns_zone_id     = azurerm_private_dns_zone.private_dns_zone["logic_app"].id
  resource_group_name       = azurerm_resource_group.rg.name
  workspace_id            = azurerm_log_analytics_workspace.app.id
  subnet_id               = azurerm_subnet.logic_app_subnet.id
  WEBSITE_DNS_SERVER      = "168.63.129.16"
  depends_on              = [azurerm_resource_group.rg, azurerm_storage_account.logic_app_state]
}
```





[Logic Apps Overview](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-overview)
[Connectors List](https://learn.microsoft.com/en-us/connectors/connector-reference/connector-reference-logicapps-connectors)
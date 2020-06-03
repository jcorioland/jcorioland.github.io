---
layout: post
title:  "Run Azure DevOps self-hosted agents in Azure Container Instance using Terraform"
date:   2020-05-12 10:00:00 +0200
categories: 
- Microsoft Azure
- Terraform
- Azure DevOps
author: 'Julien Corioland'
identifier: 'd33c206c-f679-4dcf-ad5b-f235193a633b'
image: /images/terraform-microsoft-azure-introduction/terraform-azure.png
---

Azure DevOps offers two kinds of pipeline agents: Microsoft-hosted agents, fully managed by Microsoft, and self-hosted agents, fully managed by yourself. I recently worked with a customer who needed to be able to run the pipeline agents into their private virtual network infrastructure which is one of the use case for having to use self-hosted agents.

<!--more-->

Because a lot of customers are asking for being able to spin up quickly Azure DevOps self-hosted agents, I have decided to develop a [Terraform module](https://github.com/Azure/terraform-azurerm-aci-devops-agent) that can help anyone to deploy one (or more) into a Docker container and host it quickly into [Azure Container Instance](https://azure.microsoft.com/en-us/services/container-instances/). By doing that you can have one ore more agents up and running in only a few minutes. Very useful when you have build workloads that require to scale up & down very frequently and that you want to minimize costs of hosting build agents.

## When to use Azure DevOps self-hosted agents?

In my opinion, there are only a few scenarios that justify to not use the Microsoft-managed agents:

1. You must be inside a private virtual network, control the network traffic, block the Internet access from the agent etc.
2. You are using softwares that are not available on the Microsoft-hosted agents, that cannot be installed at build time because they are too big or require GUI
3. You need the state on the agent to be maintained between two executions / job / stages, so you need to be able to target a machine explicitly
4. You require specific hardware (GPU, custom boards etc...)

If you are not in one of these cases, it should be possible for you to use any of the [Windows, Linux or macOS Microsoft-managed agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=yaml).

There are different ways to create your own self-hosted agents:

- it can be a physical or virtual machine running [Linux](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops), [Windows](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops) or [macOS](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-osx?view=azure-devops) where you install and configure the Azure DevOps agent software
- it can be a [Docker container](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)
- it can be a a set of virtual machines, living in an [Azure ScaleSet that you let Azure DevOps manage](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops) (this feature is still in preview at the time I am writing this article)

Using the [terraform-azurerm-aci-devops-agent module](https://github.com/Azure/terraform-azurerm-aci-devops-agent), the agents will be running inside Docker containers, Linux or Windows, hosted on Azure Container Instance.

## Build your own Docker image to run the DevOps agents

Before being able to run the module, you need to build your own Docker image, that contains the DevOps agents and any tools that you want to embed into the Docker containers and that you cannot install at build time. To do that, you can follow the [official documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops) or start from [the Dockerfile](https://github.com/Azure/terraform-azurerm-aci-devops-agent/blob/master/Docker/README.md) in the Terraform module repository.

Once your Docker image is ready, you need to push it into a registry. It could be the public Docker hub, the Azure Container Registry or any Docker private registry.

## Deploy the Azure DevOps agents

The [README.md](https://github.com/Azure/terraform-azurerm-aci-devops-agent/blob/master/README.md) of the module has a bunch of examples about how to use it, but let's have a simple example here. Before running this module, you need to [create an agent pool in your Azure DevOps organization](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser#creating-agent-pools) and a [personal access token](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops#permissions) that it authorized to manage this agent pool.

This configuration has 3 variables related to Azure DevOps:

- `azure_devops_org_name`: the name of your Azure DevOps organization (if you are connecting to `https://dev.azure.com/helloworld`, then `helloworld` is your organization name)
- `azure_devops_pool_name`: the name of the agent pool that you have created
- `azure_devops_personal_access_token`: the personal access token that you have generated

Let's assume that you want to deploy 5 Azure DevOps agents into a pool named `private-aci-pool` running into a dedicated virtual network.

First, you will have to create the virtual network, if it does not exists already:

```hcl
resource "azurerm_resource_group" "vnet-rg" {
  name     = "rg-aci-devops-network"
  location = "westeurope"
}

resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-aci-devops"
  address_space       = ["10.0.0.0/16"]
  location            = "westeurope"
  resource_group_name = azurerm_resource_group.vnet-rg.name
}

resource "azurerm_subnet" "aci-subnet" {
  name                 = "aci-subnet"
  resource_group_name  = azurerm_resource_group.vnet-rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]

  delegation {
    name = "acidelegation"

    service_delegation {
      name    = "Microsoft.ContainerInstance/containerGroups"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action", "Microsoft.Network/virtualNetworks/subnets/prepareNetworkPolicies/action"]
    }
  }
}
```

> Note: to allow Azure Container Instance to use the subnet, you must create [a subnet delegation](https://docs.microsoft.com/en-us/azure/virtual-network/manage-subnet-delegation). This is what is done by the `delegation` block in the code sample above.

Then, you can configure the module to create the Azure DevOps agents

```hcl
module "aci-devops-agent" {
  source                    = "Azure/aci-devops-agent/azurerm"
  enable_vnet_integration   = true
  create_new_resource_group = true
  vnet_resource_group_name  = azurerm_resource_group.vnet-rg.name
  vnet_name                 = azurerm_virtual_network.vnet.name
  subnet_name               = azurerm_subnet.aci-subnet.name
  linux_agents_configuration = {
    agent_name_prefix = "linuxagent"
    count             = 5
    docker_image      = "YOUR_DOCKER_IMAGE_NAME"
    docker_tag        = "YOUR_DOCKER_IMAGE_TAG"
    agent_pool_name   = "private-aci-pool"
    cpu               = 1
    memory            = 4
  }
  resource_group_name                = "rg-aci-devops"
  location                           = "westeurope"
  azure_devops_org_name              = "YOUR_DEVOPS_ORG_NAME"
  azure_devops_personal_access_token = "YOUR_DEVOPS_ACCESS_TOKEN"
}
```

The Terraform configuration is ready, you can deploy the agents by doing executing `terraform init` and `terraform apply`. After a few seconds / minutes (it can take bit longer for Windows containers as the Docker image to pull is bigger), you should see 5 container instances into the Azure Portal and 5 Azure DevOps agents up & running in the agents pool you have configured:

![Azure Portal - Container Instances](/images/terraform-aci-devops-agents/azure-portal-container-instances.png)

![Azure DevOps Portal - Agents](/images/terraform-aci-devops-agents/azure-devops-portal-agents.png)

> Note: you can also find some examples of use by browsing the [test cases](https://github.com/Azure/terraform-azurerm-aci-devops-agent/blob/test/fixture).

## Clean up

When you change the number of agents or call the `terraform destroy`, the container instances will be completely removed from Azure, but the agents will still be registered in the Azure DevOps pool. They have to be removed manually. This is a [known issue](https://github.com/Azure/terraform-azurerm-aci-devops-agent/issues/3). For example, if you scale down the number of agents from 5 to 3, 2 instances will be effectively removed but they will still be registered in the pool:

![Azure DevOps Portal - Agents Scale Down](/images/terraform-aci-devops-agents/azure-devops-portal-agents-scale-down.png)

## Next steps

I hope you will enjoy using this Terraform module and it will help you to deploy quickly and easily Azure DevOps agents using Azure Container Instance. Feel free to reach out on [GitHub](https://github.com/Azure/terraform-azurerm-aci-devops-agent/issues) if you encounter any issue, want to request a feature or contribute to the project!

Hope this helps!

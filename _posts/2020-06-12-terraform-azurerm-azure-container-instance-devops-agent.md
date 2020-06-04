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

Azure DevOps offers two kinds of pipeline agents: Microsoft-hosted agents, fully managed by Microsoft, and self-hosted agents. My team recently worked with a customer who needed to be able to run pipeline agents on their private virtual network infrastructure which is one of the use case for having to use self-hosted agents.

<!--more-->

A lot of customers are asking for being able to spin up quickly Azure DevOps self-hosted agents. I have decided to develop a [Terraform module](https://github.com/Azure/terraform-azurerm-aci-devops-agent) that can help anyone to deploy one (or more) into a Docker container and host it quickly into [Azure Container Instance](https://azure.microsoft.com/en-us/services/container-instances/). Following this approach you can have one ore more agents up and running in only a few minutes. Very useful when you have build workloads that require to scale up & down very frequently and that you want to minimize costs of hosting build agents.

Before going into more details, I'd like to say **thank you** my colleagues [Yuping](https://github.com/yupwei68) and [Andreas](https://github.com/aheumaier) who helped me reviewing the module and this blog post.

## When to use Azure DevOps self-hosted agents?

There are only a few scenarios that justify to not use the Microsoft-managed agents:

1. You must be inside a private virtual network, control the network traffic, block the Internet access from the agent etc.
2. You are using software that is not available on the Microsoft-hosted agents, that are too slow or too complicated to be installed at build time or require GUI
3. You need the state on the agent to be maintained between two executions / job / stages, so you need to be able to target a machine explicitly
4. You require specific hardware (GPU, custom boards etc...)

If you are not in one of these cases, it should be possible for you to use any of the [Windows, Linux or macOS Microsoft-managed agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=yaml).

There are different ways to create your own self-hosted agents:

- using a physical or virtual machine running [Linux](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops), [Windows](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops) or [macOS](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-osx?view=azure-devops) where you install and configure the Azure DevOps agent software
- using a [Docker container](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops)
- using a set of virtual machines, living in an [Azure ScaleSet that you let Azure DevOps manage](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/scale-set-agents?view=azure-devops) (this feature is still in preview at the time I am writing this article)

With the [terraform-azurerm-aci-devops-agent module](https://github.com/Azure/terraform-azurerm-aci-devops-agent), the agents will be running inside Docker containers, Linux or Windows, hosted on Azure Container Instance.

## Build our own Docker image to run the DevOps agents

Before being able to run the module, we need to build our own Docker image, that contains the DevOps agents and any tools that we want to embed into the Docker containers and that we cannot install at build time. To do that, it's possible to follow the [official documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops) or start from [the Dockerfile](https://github.com/Azure/terraform-azurerm-aci-devops-agent/blob/master/Docker/README.md) in the Terraform module repository.

Once the Docker image is ready, it needs to be pushed to a registry. It could be the public Docker hub, the Azure Container Registry or any Docker private registry.

## Deploy the Azure DevOps agents

The [README.md](https://github.com/Azure/terraform-azurerm-aci-devops-agent/blob/master/README.md) of the module has a bunch of examples about how to use it, but let's have a simple example here. Before running this module, we need to [create an agent pool in our Azure DevOps organization](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser#creating-agent-pools) and a [personal access token](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-linux?view=azure-devops#permissions) that it authorized to manage this agent pool.

This module has three variables related to Azure DevOps:

- `azure_devops_org_name`: the name of your Azure DevOps organization (if you are connecting to `https://dev.azure.com/helloworld`, then `helloworld` is your organization name)
- `azure_devops_personal_access_token`: the personal access token that you have generated
- `agent_pool_name`: both in the `linux_agents_configuration` and `windows_agents_configuration`, it is the name of the agent pool that you have created in which the Linux or Windows agents must be deployed

Let's assume that we want to deploy 5 Azure DevOps agents into a pool named `private-aci-pool` running into a dedicated virtual network.

First, we will have to create the virtual network, if it does not exists already:

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

> Note: to allow Azure Container Instance to use the subnet, we must create [a subnet delegation](https://docs.microsoft.com/en-us/azure/virtual-network/manage-subnet-delegation). This is what is done by the `delegation` block in the code sample above.

Then, we can configure the module to create the Azure DevOps agents

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
    docker_image      = "DOCKER_IMAGE_NAME"
    docker_tag        = "DOCKER_IMAGE_TAG"
    agent_pool_name   = "private-aci-pool"
    cpu               = 1
    memory            = 4
  }
  resource_group_name                = "rg-aci-devops"
  location                           = "westeurope"
  azure_devops_org_name              = "DEVOPS_ORG_NAME"
  azure_devops_personal_access_token = "DEVOPS_ACCESS_TOKEN"
}
```

The Terraform configuration is ready, we can deploy the agents by doing executing `terraform init` and `terraform apply`. After a few seconds / minutes (it can take bit longer for Windows containers as the Docker image to pull is bigger), we should see 5 container instances into the Azure Portal and 5 Azure DevOps agents up & running in the agents pool we have configured above:

![Azure Portal - Container Instances](/images/terraform-aci-devops-agents/azure-portal-container-instances.png)

![Azure DevOps Portal - Agents](/images/terraform-aci-devops-agents/azure-devops-portal-agents.png)

> Note: you can also find some examples of use by browsing the [test cases](https://github.com/Azure/terraform-azurerm-aci-devops-agent/blob/test/fixture).

## Clean up

When we change the number of agents or call the `terraform destroy`, the container instances will be completely removed from Azure, but our agents stay registered in the Azure DevOps pool. They have to be removed manually. This is a [known issue](https://github.com/Azure/terraform-azurerm-aci-devops-agent/issues/3). For example, if we scale down the number of agents from 5 to 3, 2 instances will be effectively removed but they will still be registered in the pool:

![Azure DevOps Portal - Agents Scale Down](/images/terraform-aci-devops-agents/azure-devops-portal-agents-scale-down.png)

## Next steps

I hope you enjoy using this Terraform module and it helps you to deploy Azure DevOps agents using Azure Container Instance in an easy way. Feel free to reach out on [GitHub](https://github.com/Azure/terraform-azurerm-aci-devops-agent/issues) if you encounter any issue, want to request a feature or contribute to the project!

Hope this helps!

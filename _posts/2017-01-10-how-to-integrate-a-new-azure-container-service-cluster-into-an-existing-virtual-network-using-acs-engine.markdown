---
layout: post
title:  "How to integrate a new Azure Container Service cluster into an existing virtual network using ACS Engine"
date:   2017-01-10 10:00:00 +0200
categories: 
- Azure Container Service
- Microsoft Azure
author: 'Julien Corioland'
identifier: 'd4b01e10-5b0b-49a3-95ab-e8b4975146ce'
---

When I discuss about Azure Container Service with customers, one of the most frequent question that they ask to me is "is it possible to deploy a cluster into an existing virtual network?".

And since a few weeks, since the [ACS Engine](https://github.com/Azure/acs-engine) has been released and open sourced on GitHub, I am really happy to be able to answer "yes, and it is really easy!".

<!--more-->

### What is ACS Engine?

ACS Engine, for Azure Container Service Engine, is a CLI tool that helps to generate Azure Resource Manager templates to deploy Docker enabled clusters on Microsoft Azure. It works with all the orchestrators supported by ACS: Docker Swarm, Mesosphere DC/OS and Kubernetes. Using ACS Engine to deploy a cluster instead of using the Microsoft Azure portal unlocks some really cool options like declaring multiple agent pools, with different sizes of VMs, public or private, and of course integrated the cluster into a customer virtual network that is the topic I will focus on in this post.

### How to install ACS Engine

ACS Engine is written in Go and is available for Windows, Linux and Mac OS. To be able to use it on your machine, you just have to get the source code and build it. All is documented on this [page](https://github.com/Azure/acs-engine/blob/master/docs/acsengine.md).

Once you have a built copy of ACS Engine, your are ready to go!

### Create the ACS Engine template

ACS Engine uses a JSON template in input and generates the ARM template and ARM parameters files in output.

Depending on the orchestrator you want to use, the number of agent pools, the machine size you want (etc.) this input template could differ from the one I am going to detail here. I have chosen to illustrate the custom virtual network integration feature with a Docker Swarm cluster example, but all will work in the same way for Kubernetes and DC/OS. You will find samples for those orchestrators on the [ACS-Engine Github](https://github.com/Azure/acs-engine/tree/master/examples/vnet).

I assume that you have already deployed one virtual network that contains two subnets:

- 10.100.0.0 for the master nodes
- 10.200.0.0 for the agent nodes

Here is the JSON template that I will use with the acsengine.exe CLI tool:

```json
{
    "apiVersion": "vlabs",
    "properties": {
        "orchestratorProfile": { "orchestratorType": "Swarm" },
        "masterProfile": {
            "count": 3,
            "dnsPrefix": "swarmcustommgmt",
            "vmSize": "Standard_D2_v2",
            "vnetSubnetId":     "/subscriptions/REPLACE_WITH_SUB_ID/resourceGroups/REPLACE_WITH_RESOURCE_GROUPE_NAME/providers/Microsoft.Network/virtualNetworks/REPLACE_WITH_VNET_NAME/subnets/SwarmMaster",
            "firstConsecutiveStaticIP": "10.100.0.1"
        },
        "agentPoolProfiles": [
            {
                "name": "agentprivate",
                "count": 2,
                "vmSize": "Standard_D3_v2",
                "vnetSubnetId":     "/subscriptions/REPLACE_WITH_SUB_ID/resourceGroups/REPLACE_WITH_RESOURCE_GROUPE_NAME/providers/Microsoft.Network/virtualNetworks/REPLACE_WITH_VNET_NAME/subnets/SwarmNode"
            },
            {
                "name": "agentpublic",
                "count": 2,
                "vmSize": "Standard_D2_v2",
                "dnsPrefix": "swarmpublicagent",
                "vnetSubnetId":     "/subscriptions/REPLACE_WITH_SUB_ID/resourceGroups/REPLACE_WITH_RESOURCE_GROUPE_NAME/providers/Microsoft.Network/virtualNetworks/REPLACE_WITH_VNET_NAME/subnets/SwarmNode",
                "ports": [ 80, 443 ]
            }
        ],
        "linuxProfile": {
            "adminUsername": "azureuser",
            "ssh": {
                "publicKeys": [
                    {
                        "keyData": "REPLACE_WITH_YOUR_SSH_PUBLIC_KEY"
                    }
                ]
            }
        }
    }
}
```

As you can see, this file is really close to the Azure Resource Manager JSON format. It contains a collection of properties that define the cluster you will create. You can set the type of the orchestrator you want to use using the `orchestratorType` property. Then, you can define the profile and number of virtual machine you want for the Docker Swarm masters (here 3 Standard_D2_V2 virtual machines) and also the different agent pools you want to create.

In this case, I create two different agent pools, each compose by two virtual machines, one private (which is not connected to any Azure load balancer or public IP) and one public with 2 load balancing rules on HTTP 80 and 443 ports.

Each agent node definition and the master node definition contains a property `vnetSubnetId` that represents the identifier of the virtual network where you want to deploy the cluster.

Finally, you need to specify the SSH public key that will be used to secure the connection to your cluster.

Once you have defined this JSON template to fit with your needs, there is only one command to execute:

```bash
acs-engine "PATH TO YOUR INPUT JSON FILE"
```

This command will generate two JSON files:

- azuredeploy.json
- azuredeploy.parameters.json

![ACS-Engine Custom VNET](/images/msdn-archives/acs-engine-custom-vnet-01.png)

These two files can now be used to deploy the cluster, using PowerShell:

```PowerShell
New-AzureRmResourceGroupDeployment -Name CustomVNETSwarmDeployment -ResourceGroupName REPLACE_WITH_RESOURCE_GROUPE_NAME -TemplateFile .\azuredeploy.json -TemplateParameterFile .\azuredeploy.parameters.json
```

Or using the Azure CLI:

```PowerShell
azure group deployment create -f "azuredeploy.json" -e "azuredeploy.parameters.json" -g REPLACE_WITH_RESOURCE_GROUPE_NAME -n CustomVNETSwarmDeployment
```

And voilà! Your cluster is going to be deployed in the existing virtual network.

Enjoy !
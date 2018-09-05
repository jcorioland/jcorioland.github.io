---
layout: post
title:  "Azure Container Service from the Azure CLI"
date:   2016-07-21 10:00:00 +0200
categories: 
- Docker
- Microsoft Azure
- Azure Container Service
author: 'Julien Corioland'
identifier: '91c5ebe3-0e45-4187-bb1a-105c48b838b4'
---

Azure CLI is a node.js cross-platform tool that runs on Linux, Mac OS and Windows and allows to work with Microsoft Azure from the command line. The latest version of Azure CLI includes some commands to create and manage [Azure Container Service](https://azure.microsoft.com/en-us/services/container-service/) clusters.

<!--more-->

### Install Azure CLI

Azure CLI can be installed using the node.js package manager (npm):

```bash
npm install –g azure-cli
```

If you already have installed this tool, you may have to upgrade to the latest version using:

```bash
npm update –g azure–cli
```

The latest published version is 0.10.2. You can check your current version with the command:

```bash
azure ––version
```

You are now ready to work with Azure CLI.

### Connect to your Microsoft Azure subscription

To connect your Microsoft Azure account with Azure CLI, use the `azure login` command. It will generates a code to use on [http://aka.ms/devicelogin](http://aka.ms/devicelogin). Open this URL in your web browser and enter the code. Then log in to your Microsoft Azure account. Once done, your subscriptions will be imported and usable from the Azure CLI:

![Azure CLI](/images/msdn-archives/acs-from-azure-cli-01.png)

![Azure CLI](/images/msdn-archives/acs-from-azure-cli-02.png)

![Azure CLI](/images/msdn-archives/acs-from-azure-cli-03.png)

Azure CLI supports both ASM (Azure Service Management) and ARM (Azure Resource Management) modes. Azure Container Service requires that you switch to the ARM mode. To do that, use the command:

```bash
azure config mode arm
```

### Create an Azure Container Service cluster with Azure CLI

First, you need to generate a JSON parameters file that will be used for the deployment. The generation can be done using the command:

```bash
azure acs config create ––json ––parameter-file NameOfYourJSONConfigFile.json
```

Open your favorite JSON editor (I am personally using Visual Studio Code which is pretty cool). As you can see, all the structure is here, you just have to complete some parameters:

![Azure CLI](/images/msdn-archives/acs-from-azure-cli-04.png)

The first JSON object is about the orchestrator you want to use. You need to complete the `orchestratorType` value with either `Swarm` or `DCOS` depending on the orchestrator you want to deploy in your cluster. In this example, I have chosen to use Docker Swarm:

```json
"orchestratorProfile": { "orchestratorType": "Swarm" }
```

Then you need to complete parameters for the master nodes availability set. You have to specify the number of masters that you want to deploy (1-3-5), the DNS prefix and FQDN where the masters will be accessible. The FQDN is the concatenation of the prefix and `your_deployment_region.cloudapp.azure.com`. In this case I want 1 master node, deployed on acsazurecli-master.northeurope.cloudapp.azure.com:

```json
"masterProfile": { "count": 1, "dnsPrefix": "acsazurecli-master", "fqdn": "acsazurecli-master.northeurope.cloudapp.azure.com" }
```

You need also configure the agent pools with the number of machines you want in your cluster, the size of these machines and the DNS/FQDN information:

```json
"agentPoolProfiles": [ { "name": "agentpools", "count": 2, "vmSize": "Standard_A2", "dnsPrefix": "acsazurecli-agents", "fqdn": "acsazurecli-agents.northeurope.cloudapp.azure.com" } ]
```

For the Docker Swarm deployment, there are no Windows jumpbox so you need to remove the windowsProfile JSON section.
You need to provide are the username and the SSH public key that will be used to SSH to the master node. You can read this blog post to learn how to generate an SSH key. Once generated, just past the content of the public key in the keyDate JSON property:

```json
"linuxProfile": { "adminUsername": "jcorioland", "ssh": { "publicKeys": [ { "keyData": "ssh-rsa YOUR PUBLIC SSH PUBLIC KEY" } ] } }
```

And finally, update the location value at the bottom of the file with the Microsoft Azure location where you want to deploy the cluster:

```json
"location": "northeurope"
```

Save your file and you’re done for the configuration step!

> Note: if you do not want to edit the file manually, you can use `acs config` commands from Azure CLI to configure each section:

![Azure CLI](/images/msdn-archives/acs-from-azure-cli-05.png)

Now that your parameters file is ready, you can create a new resource group using the following command:

```bash
azure group create YourResourceGroupName RegionWhereYouWantToCreateTheGroup
```

I have chosen to create a resource group named "AzureCliAcs-RG" in the North Europe region:

```bash
azure group create AzureCliAcs-RG northeurope
```

Finally, use the `azure acs create` command to create a new cluster based on your parameters files:

```bash
azure acs create ––json ––parameter-file NameOfYourParameterFile.json --resource-group YourResourceGroupName --name NameOfYourContainerService
```

Wait for the deployment to be completed (it can take a while depending on the number of agents you asked for).
Once the deployment has completed, you can get information about the Azure Container Service cluster using the command:

```bash
azure acs show YourResourceGroupName NameOfYourContainerService
```

### Scale the number of agents in your Azure Container Service cluster

Another cool command that has been implemented in the Azure CLI ACS is the scale command. It allows to create more nodes in you cluster if you need to increase its capacity. To do that, you simply need to use the azure acs scale command. For example if you want 4 agents in the cluster type:

```bash
azure acs scale YourResourceGroupName NameOfYourContainerService 4
```

Very cool, isn’t it ? To see all the features implemented for Azure Container Service in the Azure CLI tool, you can use the `–h` parameter on the azure acs command.
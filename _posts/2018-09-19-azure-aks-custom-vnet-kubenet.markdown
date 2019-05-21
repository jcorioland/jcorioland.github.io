---
layout: post
title:  "How to: Azure Kubernetes Service + Custom VNET with Kubenet"
date:   2018-09-19 16:00:00 +0200
categories: 
- Microsoft Azure
- Kubernetes
author: 'Julien Corioland'
identifier: '30c7ccf5-38c9-45ad-8218-523ca6c8ad20'
---

*05/21/2019 UPDATE: the route table and NSG assignation are now directly managed by the Azure Kubernetes Service provider, you don't need to run extra script anymore! This blog post has been updated according to this and this kind of deployment is now documented on Microsoft Azure docs, on [this page](https://docs.microsoft.com/en-us/azure/aks/configure-kubenet).*

You probably already know that it is possible to deploy an Azure Kubernetes Service cluster into an existing virtual network (VNET) to be able to control the network CIDR and consume other services on your private networks, like on-premises services through an [Express Route](https://azure.microsoft.com/en-us/services/expressroute/) for example.

If you read the [network documentation of AKS](https://docs.microsoft.com/en-us/azure/aks/networking-overview) you will see that there are two networking modes: **Basic networking** that controls the virtual network and uses [Kubenet](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#kubenet) as network plugin or **Advanced networking** that lets you control the virtual network and uses [Azure CNI](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md) network plugin.

In this post, I will explain how to use Advanced networking, to keep control on the virtual network, but continue to use Kubenet as a network plugin.

> *Thank you very much to my colleague [Stéphane Erbrech](https://twitter.com/serbrech) who has helped a lot to get this scenario working and review this post before publication*

<!--more-->

A lot of the customers I am working with need to control the virtual network in which the Azure Kubernetes Service cluster should be deployed, most of the time for being able to peer this virtual network with their existing infrastructure and enable on premise communications. Azure CNI makes sure that all your pods running in the Kubernetes cluster get an IP address on the subnet, which is great because you can benefit from all the VNET/Subnet features and security rules using NSGs. But sometime it can be complicated to have one IP address per pod, for example when you have to deal with very small IP addresses range on the subnet where AKS will land. In this case, it is really interesting to keep Kubenet (that comes with a private network inside the cluster).

This scenario is not supported using Azure CLI or Azure Portal but we have published all the scripts to get ready on [this GitHub repository](https://github.com/serbrech/AKS-vnet-kubenet). Let's have a look at how this sample works!

### Environment variables

After cloning the repository, you need to update the `scripts/env.sh` file. It contains all the environment variables that will be used by the deployment scripts.

First, fill your Azure subscription id and [service principal](https://docs.microsoft.com/en-us/azure/aks/kubernetes-service-principal) information and uncomment the exports.

```bash
export AKS_LOCATION=eastus

##### EDIT THESE ######
#
# If you don't have a SPN, uncomment the line that creates it in the setup-aks-vnet.sh script
#
# export SPN_PW=<YOUR_SPN_PW> # a service principal Service Principal
# export SPN_CLIENT_ID=<YOUR_SPN_CLIENT_ID>

# export AKS_SUB=<YOUR_AZURE_SUB_ID>
```

Azure Kubernetes proposes [native integration with OMS Container Monitoring Solution](https://docs.microsoft.com/en-us/azure/monitoring/monitoring-container-health?toc=%2Fen-us%2Fazure%2Faks%2FTOC.json&bc=%2Fen-us%2Fazure%2Fbread%2Ftoc.json). You need to fill the name, id and location of the OMS workspace you want to use in the `OMS_WORKSPACE_NAME`, `OMS_WORKSPACE_ID` and `OMS_LOCATION` variables.

```bash
# export OMS_WORKSPACE_NAME=<WORKSPACE_NAME>
# export OMS_WORKSPACE_ID=<WORKSPACE_ID> # /subscriptions/<subId>/resourcegroups/<om_srg>/providers/Microsoft.OperationalInsights/workspaces/<OMS_WORKSPACE_NAME>
# export OMS_LOCATION=${AKS_LOCATION}
```

> Note: The `AKS_DATE` variable is a generated suffix for the resources names. If you don't need that, just remove it or fix it to what you want.

```bash
export AKS_DATE=$(date +%Y%m%d)f

export AKS_RG=aks-${AKS_DATE}
export AKS_NAME=aks-${AKS_DATE}
```

If the virtual network and the subnet where AKS has to be deployed already exist, you need to set the `AKS_VNET_RG`, `AKS_VNET_LOCATION` and `AKS_VNET_NAME` environment variables values to the name of the resource group that contains the virtual network, the location of the virtual network and the name of the virtual network, respectively. You also need to update the `AKS_VNET_RANGE`, `AKS_SUBNET2_NAME` and `AKS_SUBNET2_RANGE` with the range of the virtual network, the range of the subnet and the subnet name.

```bash
export AKS_VNET_RG=aks-shared-${AKS_DATE}
export AKS_VNET_LOCATION=${AKS_LOCATION}
export AKS_VNET_NAME=aks-vnet-${AKS_VNET_LOCATION}

export AKS_VNET_RANGE=10.201.0.0/16
export AKS_SUBNET1_NAME=VNET-Local
export AKS_SUBNET1_RANGE=10.201.0.0/22
export AKS_SUBNET2_NAME=AKS-Nodes
export AKS_SUBNET2_RANGE=10.201.4.0/22
```

> If the virtual network already exists, you probably won't need the `AKS_SUBNET1_*`, just remove them.

AKS cluster will be deployed inside the subnet referenced by the `AKS_SUBNET` variable. We also let you the option to choose the Docker bridge address, the Kubernetes DNS service IP address and the Kubernetes service address range, as documented [here](https://docs.microsoft.com/en-us/azure/aks/networking-overview#plan-ip-addressing-for-your-cluster). But, most of the time, you don't need to update this configuration.

```bash
####
#### NOTE : changing the CIDR or the DNS IP will make your unable to scale your cluster if you cli is not up to date.
#### make sure your Azure CLI is at least in version 2.0.37  
####
export AKS_SUBNET=/subscriptions/${AKS_SUB}/resourceGroups/${AKS_VNET_RG}/providers/Microsoft.Network/virtualNetworks/${AKS_VNET_NAME}/subnets/${AKS_SUBNET2_NAME}
# export AKS_BRIDGE_IP=10.201.0.1/24
export AKS_DNS_IP=10.0.0.10 # Must be within SVC_CIDR
export AKS_SVC_CIDR=10.0.0.0/16 # Must not overlap with AKS_VNET_RANGE
```

### Deployment script

Now that your environment variables are configured, you can jump to the `scripts/deploy-aks-custom-vnet.sh` script that is responsible for deploying the AKS cluster.

If you did not provide Service Principal credentials in the `env.sh` script, uncomment the two lines that are creating a new one and retrieving its information for you:

```bash
create a SP and sets the env variable accordingly
eval $(az ad sp create-for-rbac --skip-assignment | jq -r '"export SPN_PW=\(.password) && export SPN_CLIENT_ID=\(.appId)"')
```

If you are deploying in an existing virtual network you can remove the first line, that creates the resource group for the VNET:

```bash
az group create -n ${AKS_VNET_RG} -l ${AKS_VNET_LOCATION}
```

You can also remove the second command `az network vnet create`:

```bash
az network vnet create \
--location ${AKS_VNET_LOCATION} \
-g ${AKS_VNET_RG} \
--name ${AKS_VNET_NAME} \
--address-prefixes ${AKS_VNET_RANGE} \
--subnet-name ${AKS_SUBNET1_NAME} \
--subnet-prefix ${AKS_SUBNET1_RANGE}
```

If the subnet where you want to deploy AKS already exists, remove the third command in the scripts: `az network vnet subnet create`:

```bash
az network vnet subnet create \
-g ${AKS_VNET_RG} \
--name ${AKS_SUBNET2_NAME} \
--address-prefix ${AKS_SUBNET2_RANGE} \
--vnet-name ${AKS_VNET_NAME}
```

Then we make sure that the Service Principal has `Contributor` role on the resource group that contains the virtual network:

```bash
az role assignment create \
--role=Contributor \
--scope=/subscriptions/${AKS_SUB}/resourceGroups/${AKS_VNET_RG} \
--assignee ${SPN_CLIENT_ID}
```

> *Note: if you want more control, you can set the scope to the subnet resource id and not the whole resource group, it will also work. This role assignment is required as Kubernetes will use the service principal to create external/internal load balancers for your published services*

The next step create the deployment using the `aks-vnet-all.json` ARM template, after overriding some parameters:

```bash
# Create a deployment from a local template, using a parameter file and selectively overriding key/value pairs.
az group deployment create -g ${AKS_RG} \
--template-file aks-vnet-all.json \
--parameters @aks-vnet-all-parameters.json \
--parameters \
    resourceName=${AKS_NAME} \
    location=${AKS_VNET_LOCATION} \
    networkPlugin=kubenet \
    vnetSubnetID="/subscriptions/${AKS_SUB}/resourceGroups/${AKS_VNET_RG}/providers/Microsoft.Network/virtualNetworks/${AKS_VNET_NAME}/subnets/${AKS_SUBNET2_NAME}" \
    servicePrincipalClientId=${SPN_CLIENT_ID} \
    servicePrincipalClientSecret=${SPN_PW} \
    workspaceRegion=${OMS_LOCATION} \
    workspaceName=${OMS_WORKSPACE_NAME} \
    omsWorkspaceId=${OMS_WORKSPACE_ID} \
    serviceCidr=${AKS_SVC_CIDR} \
    dnsServiceIP=${AKS_DNS_IP} \
    enableRbac=true
```

You may also want to override some default values in the `aks-vnet-all-parameters.json` parameters file. For example if you want to enable http application routing, disable RBAC or OMS integration, change Kubernetes version... Like with the CLI or the Azure portal, you can also update the number of nodes that you want in the cluster, the size of the virtual machines etc...

Once you have updated the script following your needs, you are ready to deploy:

```bash
./deploy-aks-custom-vnet.sh
```

Wait for the deployment to be completed.

### Network fix

There is currently an issue while deploying AKS with Advanced Networking and Kubenet plugin: the route table and NSG created in the `MC_*` resource group are not associated to the subnet where the cluster is deployed. To make sure the network will work while the cluster is deployed, the script will do this association for you, right after the deployment:

```bash
# We need to update the VNET with the Route Table and NSG from AKS
# First, find the resources we need : AKS RG, Route table, NSG and subnet id.     
AKS_MC_RG=$(az group list --query "[?starts_with(name, 'MC_${AKS_RG}')].name | [0]" --output tsv)
ROUTE_TABLE=$(az network route-table list -g ${AKS_MC_RG} --query "[].id | [0]" -o tsv)
AKS_NODE_SUBNET_ID=$(az network vnet subnet show -g ${AKS_VNET_RG} --name ${AKS_SUBNET2_NAME} --vnet-name ${AKS_VNET_NAME} --query id -o tsv)
AKS_NODE_NSG=$(az network nsg list -g ${AKS_MC_RG} --query "[].id | [0]" -o tsv)

# Update the Subnet
az network vnet subnet update \
-g $AKS_VNET_RG \
--route-table $ROUTE_TABLE \
--network-security-group $AKS_NODE_NSG \
--ids $AKS_NODE_SUBNET_ID
```

> If you want remove the cluster, you need to manually remove the NSG and route table associations first.

Once the deployment is completed you'll get a Kubernetes cluster in a custom vnet, but using the Kubenet plugin. Then you can use this cluster as any AKS cluster.

Hope this helps!
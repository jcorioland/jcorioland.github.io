---
layout: post
title:  "Building a cloud native platform on top of Microsoft Azure"
date:   2021-05-01 10:00:00 +0200
categories: 
- DevOps
- Kubernetes
- Microsoft Azure
author: 'Julien Corioland'
identifier: 'af64746b-9155-4fc0-b596-8a5dd7232422'
image: /images/todo
---

Over the past 8 months, my team and I have been engaged with a large ISV to help build and run a cloud native platform on top of Microsoft Azure. The same platform can also run on-premises and on other public clouds. 

In this article, I will share some learnings and thoughts about different items that might be useful for you if you are starting on such a project, so you can get the best of Microsoft Azure to build cloud native platform for your company application developers and operators. I will not detail everything about the implementation itself, but stay at the architecture overview level of the discussion. That's being said, the following contains a lot of external links to specific topics, if you want to go deeper into each of them.

Let's start by understanding what is a cloud native platform. 

<!--more-->

## Cloud Native Platform, what does it mean?

The Cloud Native Computing Foundation (CNCF) provides [one official definition](https://github.com/cncf/foundation/blob/master/charter.md) of what means "cloud native":

*Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.*

*These techniques enable loosely coupled systems that are resilient, manageable, and observable. Combined with robust automation, they allow engineers to make high-impact changes frequently and predictably with minimal toil.*

The idea behind this is to try to codify and unify the way to build, deploy and run enterprise applications. There are more and more companies trying to build a consistent platform across clouds (public, private or hybrid) to ensure they use the same tools and process to deploy all their applications. As an application developer or operator, you don't want to care about where the application runs it could be in a private or public cloud. You don't want to have to implement behaviors in the applications because it can run in different places. As well as a DevOps, you don't want to have different way of packaging, deploying and operating applications across the company, depending on where they are actually running.

Building a platform that targets multiple clouds is not an easy task. It's a trade off between going with full managed 1st party technologies (most of the time available as PaaS/SaaS, so easy to use / operate), and cross-platforms technologies coming from the [cloud-native landscape](https://landscape.cncf.io/). The ultimate goal is to abstract the underlying platform from your end users (applications developers and operators), but also make sure you do not take any architecture decision that could be influenced by how you designed something on another cloud or on-premise.

No doubt that Kubernetes is the central piece of cloud-native platform these days, as it is designed to provide abstraction of the infrastructure on top of which it is running. Kubernetes comes with a complete deployment and development model to ensure that applications running in Kubernetes do not have to know or interact with the actual machines they are running on, or about network, persistent storage, security boundaries they are using.

The goal of this article is not to go into details about the application layer itself and how to deploy applications inside a Kubernetes cluster. There is already tons of articles and documentations about that. If you want to know more about how to get started with application development/deployment with Kubernetes on Azure, you can start from [there](https://docs.microsoft.com/azure/aks/concepts-clusters-workloads).

What I am interested in sharing today is the hidden side of the iceberg: how it's possible to build on top of Azure Kubernetes Service (AKS) and provide to internal or external customers a platform where they can deploy applications, without even thinking or knowing that it actually runs on Azure. Actually, it's not only about AKS. It's about how it's possible to automate the provisioning of an enterprise-grade, high scale, secured Kubernetes on Azure.

But first thing first... let's go through about some basics.

## Landing Zone

All begins with a landing zone. This is the basic infrastructure where the platform will be deployed. It includes components such as Azure subscription, network topology and connectivity (express route, firewall, vnet, route tables, dns configuration...), identity and role-based access control, business continuity and disaster recovery.

Read more about Azure Landing Zone [here](https://docs.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/), part of the Cloud Adoption Framework documentation.

Landing Zones are mostly about automation with intensive use of infrastructure as code (ARM template, Terraform, Bicep...) to make sure it's possible to create one quickly, in an automated and reproductible way.

There are [different types of landing zones implementations documented in the Cloud Adoption Framework](https://docs.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/implementation-options#implementation-options). In the context of the project, we developed something similar to [hybrid connectivity with hub and spoke](https://github.com/Azure/Enterprise-Scale/blob/main/docs/reference/adventureworks/README.md):

![Hub & Spoke Landing Zone](/images/building-cloud-native-platform-microsoft-azure/hub-spoke-landing-zone.jpg)

Zooming on the network topology of one landing zone could lead to something like the following:

![Landing Zone Network Topology](/images/building-cloud-native-platform-microsoft-azure/landing-zone-network.jpg)

Obviously, the network topology will vary depending on your needs, but the principles will be the same. For this case study, let's consider we have the following network topology. Each and every "cloud native platform" landing zone define a virtual network with a peering connection to the hub. This virtual network contains a bunch of subnets:
- **Access subnet**: this is the only subnet that is accessible from other spokes / landing zone. It is the network entry point of the landing zone. It can contain jumbox server, Azure DevOps/Jenkins agents etc.
- **Integration subnet**: this is the subnet where private endpoints from "external" managed service will land. For example, if you are using Azure Storage, Azure Container Registry or Azure KeyVault - these services can get a private IP address on the landing zone virtual network, in this integration subnet.
- **Backing Services Subnet**: this is where any backing services (data stores, caches, message brokers) can be deployed.
- **AKS Subnet**: this is the subnet or subnets (if multiple clusters deployment) that will be used by Azure Kubernetes Service.
- **Other Subnet**: any other subnet you need in the landing zone. 

## Azure Kubernetes Service

### Private Cluster Deployment

The landing zone is ready, let's talk about Kubernetes. In this kind of cloud / network topology, it's a common practice to deploy Azure Kubernetes Service using the [private cluster feature](https://docs.microsoft.com/azure/aks/private-clusters). It will make sure that the cluster has not public endpoint and is reachable only from a subnet in the landing zone virtual network.

When you chose this deployment mode for AKS, you end with:
- no public endpoint for the cluster-api (the endpoint that is used by `kubectl` to manage the cluster)
- a private endpoint with a private IP address on the subnet targeted by the AKS deployment (use one subnet per AKS cluster)

Deploying a private AKS cluster does not mean that you cannot have a public endpoint for your applications running in the cluster. It means that the Kubernetes management endpoint (api server) is private and reachable only from a private endpoint in a given subnet.
In addition to this private cluster deployment, you might have additional network rules that forces the traffic to go through a firewall (in Azure or on-premises) or block Internet access for some resources. Azure Kubernetes Service requires that you open access to some well-known resources in order to be able to intialize and pull some container images, security patches etc. All you need to know about restricting AKS egress traffic is available [here](https://docs.microsoft.com/azure/aks/limit-egress-traffic).

### Cluster identities and separation of concerns

Identity management is an important piece of your platform. When working with Azure Kubernetes Service, you can apply separation of concerns principles by providing different identities for different puroposes, the most common being:

- AKS Cluster Identity
- Kubelet identity
- Azure Active Directory Pod identity

There are other use of managed identities with AKS, depending on your scenario and the add-ons you are using. You can find some documentation on [this page](https://docs.microsoft.com/azure/aks/use-managed-identity).

#### AKS Cluster Identity

This is the identity that will be used by Azure Kubernetes Service for all task related to Azure-platform integration, like VNET integration, route table / NSG manipulation, Azure Disk/Storage operations for persistent volumes etc.
This identity can be a Service Principal, a User-Assigned Managed identity or a System Identity. In the first option, you can let AKS create it for you or bring your own-one.

#### Kubelet identity

By default, Kubelet will use the identity provided for the AKS cluster for pulling images from an Azure Container Registry. In some cases, you might want to overrides the Kubelet identity to give an identity that is authorized to access only an ACR, and that has no other permissions required by AKS (like Storage or VNET integration). You can use the [Bring Your Own MI for Kubelet](https://docs.microsoft.com/azure/aks/use-managed-identity#bring-your-own-kubelet-mi-preview) to do that.

#### Azure Active Directory Pod Identity

[Azure Active Directory Pod Identity](https://docs.microsoft.com/azure/aks/use-azure-ad-pod-identity) is a component that you can run inside an Azure Kubernetes Service cluster to enable your applications to use managed identity to access external Azure resources. Imagine that you have a Web API running in Kubernetes that needs to access an Azure SQL Database. You don't want to store credentials in Kubernetes secrets or environment variable. Pod-identity will allow the pod running your application to work with managed identities and access the SQL Database.

This component can also be used to authenticated the Secret Store CSI Driver and let [Kubernetes read secrets directly from an Azure KeyVault](#keyvault-secret-management).

### Azure Active Directory Integration

[Azure Active Directory integration](https://docs.microsoft.com/azure/aks/managed-aad) is a different concept from cluster identities described above. It is related to how your Kubernetes users will connect and interact with the cluster and cluster API. It's possible to integrate Azure Kubernetes Service with Azure Active Directory and make sure that you manage authentication and authorization in the cluster using Azure Role-Based Access Control (RBAC).

Once configured, you can use principals object ids or e-mails and Azure AD groups object ids to define Kubernetes-RBAC rules to allow this or that user to access a given namespace or resource inside the cluster. This means that the first time a user will do a `kubectl` on a AD-enabled AKS cluster, he will be asked to connect to they Azure account.
For everything related to non-interactive login, like your automation pipeline deploying applications inside Kubernetes, you can use [kubelogin](https://docs.microsoft.com/azure/aks/managed-aad#non-interactive-sign-in-with-kubelogin).

Instead of writing your own Kubernetes-RBAC manifest referencing users and group object ids, you might also want to enable [Azure RBAC support for Kubernetes](https://docs.microsoft.com/azure/aks/manage-azure-rbac), which let you assign Kubernetes-specific roles to Azure principals and groups.

### Kubernetes cluster design

This section will discuss about the design of the Kubernetes cluster itself, especially about node pools, availability zones support, scaling/auto-scaling operations etc.
**TODO**

### KeyVault Secret Management

Your applications running in Kubernetes need secrets.  
https://docs.microsoft.com/azure/aks/csi-secrets-store-driver
**TODO**

## Observability

Observability is a central piece of your cloud native platform. There are different ways to implement observability on Azure. You can go with the Azure built-in stack like Azure Monitor, Logs Analytics etc. Or you can decide to plug external tools like Prometheus, Grafana, Elastic Search, Logstash, Kibana...

If you build an Azure-only platform, I would recommend to go with the first option, as you will get everything you need very easily. If you are building a platform across multiple cloud then the second option is probably better for you. You want to have the same dashboards, the same alerts, and the same observability practices for your platform even if they are running at different places.

### Azure Monitor - The Azure built-in way
Azure Monitor + Container Insights
**TODO**

### Prometheus, Grafana, ELK...
**TODO**

#### Azure Monitor integration with Prometheus/Grafana

- Native integration between Grafana and Azure Monitor: https://grafana.com/grafana/plugins/grafana-azure-monitor-datasource/ - Grafana connects directly to Azure Monitor data, no need for Prometheus in the middle
- If you need ELK to be the "source of truth", aggreating all metrics from all platforms from any cloud, then you can use [Metricbeat](https://www.elastic.co/guide/en/beats/metricbeat/7.10/metricbeat-module-azure.html) to extract from Azure Monitor and import to Elastic Search
- If you need Prometheus to be the "source of truth", agreggating all metrics from all platforms from any cloud, then you can use [Promitor](https://promitor.io) to scrap automatically metrics from Azure monitor and import them into Prometheus.

**TODO**

#### Azure Logs Streaming

Azure has standard support for streaming the logs of every service into a dedicated storage account or [Event Hubs](https://docs.microsoft.com/azure/active-directory/reports-monitoring/tutorial-azure-monitor-stream-logs-to-event-hub).  
From this, it's possible to use a components like [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/7.10/filebeat-module-azure.html) that will take the Event Hubs stream has input and will output it into Elastic Search.

You can read more about [Azure observability with Elastic on this page.](https://www.elastic.co/blog/monitoring-azure-infrastructure-with-filebeat-and-elastic-observability)

**TODO**

## Cost optimization

Turning ON/OFF Environment
Spot VMs
**TODO**

## Conclusion
**TODO**


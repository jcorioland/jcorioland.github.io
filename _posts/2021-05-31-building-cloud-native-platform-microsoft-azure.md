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

Over the past 8 months, my team and I have been engaged with a large ISV to help build and run a cloud native platform on top of Microsoft Azure. The same platform can also run on-premises and on other public clouds. Building a platform that targets multiple clouds is not an easy task. It's a trade off between going with full managed 1st party technologies (most of the time available as PaaS/SaaS, so easy to use / operate), and cross-platforms technologies coming from the [cloud-native landscape](https://landscape.cncf.io/). The ultimate goal is to abstract the underlying platform from your end users (applications developers and operators), but also make sure you do not take any architecture decision that could be influenced by how you designed something on another cloud or on-premise.

In this article, I will share some learnings and thoughts about different items that might be useful for you if you are starting on such a project, so you can get the best of Microsoft Azure to build cloud native platform for your company application developers and operators. I will not detail everything about the implementation itself, but stay at the architecture overview level of the discussion. That's being said, the following contains a lot of external links to specific topics, if you want to go deeper into each of them. 

<!--more-->

## Cloud Native Platform, what does it mean?

The Cloud Native Computing Foundation (CNCF) provides [one official definition](https://github.com/cncf/foundation/blob/master/charter.md) of what means "cloud native":

*Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.*

*These techniques enable loosely coupled systems that are resilient, manageable, and observable. Combined with robust automation, they allow engineers to make high-impact changes frequently and predictably with minimal toil.*

The idea behind this is to try to codify and unify the way to build, deploy and run applications in the enterprise. And they are more and more company trying to build a consistent platform across cloud / hybrid cloud to ensure they use the same tools and process to deploy all their applications. As an application developer or operator, you don't want to care about where the application runs it could be in a private or public cloud. You don't want to have to implement behaviors in the applications because it can run in different places. As well as a DevOps, you don't want to have different way of packaging, deploying and operating applications across the company, depending on where they are actually running.

No doubt that Kubernetes is the central piece of cloud-native platform these days, as it is designed to provide abstraction of the infrastructure on top of which it is running. Kubernetes comes with a complete deployment and development model to ensure that applications running in Kubernetes do not have to know or interact with the actual machines they are running on, or about network, persistent storage, security boundaries they are using.

The goal of this article is not to go into details about the application layer itself and how to deploy applications inside a Kubernetes cluster. There is already tons of articles and documentations about that. If you want to know more about how to get started with application development/deployment with Kubernetes on Azure, you can start from [there](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads).

What I am interested in sharing today is the hidden side of the iceberg: how it's possible to build on top of Azure Kubernetes Service (AKS) and provide to internal or external customers a platform where they can deploy applications, without even thinking or knowing that it actually runs on Azure. Actually, it's not only about AKS. It's about how it's possible to automate the provisioning of an enterprise-grade, high scale, secured Kubernetes on Azure.

But first thing first... let's go through about some basics.

## Landing Zone

All begins with a landing zone. This is the basic infrastructure where the platform will be deployed. It includes components such as Azure subscription, network topology and connectivity (express route, firewall, vnet, route tables, dns configuration...), identity and role-based access control, business continuity and disaster recovery.

Read more about Azure Landing Zone [here](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/), part of the Cloud Adoption Framework documentation.

Landing Zones are mostly about automation with intensive use of infrastructure as code (ARM template, Terraform, Bicep...) to make sure it's possible to create one quickly, in an automated and reproductible way.

There are [different types of landing zones implementations documented in the Cloud Adoption Framework](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/implementation-options#implementation-options). In the context of the project, we developed something similar to [hybrid connectivity with hub and spoke](https://github.com/Azure/Enterprise-Scale/blob/main/docs/reference/adventureworks/README.md):

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

The landing zone is ready, let's talk about Kubernetes. In this kind of cloud / network topology, it's a common practice to deploy Azure Kubernetes Service using the [private cluster feature](https://docs.microsoft.com/en-us/azure/aks/private-clusters). It will make sure that the cluster has not public endpoint and is reachable only from a subnet in the landing zone virtual network.

When you chose this deployment mode for AKS, you end with:
- no public endpoint for the cluster-api (the endpoint that is used by `kubectl` to manage the cluster)
- a private endpoint with a private IP address on the subnet targeted by the AKS deployment (use one subnet per AKS cluster)

### High availability, High Scale

#### Support for availability zone
#### Cluster-Autoscaler

### Azure Active Directory Integration

https://docs.microsoft.com/en-us/azure/aks/managed-aad

### KeyVault Secret Management

https://docs.microsoft.com/en-us/azure/aks/csi-secrets-store-driver

## Automation, DevOps, Continuous Deployment

Jenkins/AzDo
Infrastructure as code: Terraform, ARM
Configuration Management
Continous Deployment / GitOps

## Logging and Monitoring

Prometheus + Promitor, Grafana, ELK + FileBeat
Diagnostics streaming to Event Hubs

## Cost optimization

Turning ON/OFF Environment
Spot VMs

## Conclusion


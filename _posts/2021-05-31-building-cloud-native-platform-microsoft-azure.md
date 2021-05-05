---
layout: post
title:  "Building cloud native platform on top of Microsoft Azure"
date:   2021-05-31 10:00:00 +0200
categories: 
- DevOps
- Kubernetes
- Microsoft Azure
author: 'Julien Corioland'
identifier: 'af64746b-9155-4fc0-b596-8a5dd7232422'
image: /images/todo
---

Over the past 8 months, my team and I have been engaged with a large ISV to help them to build and run a cloud native platform on top of Microsoft Azure. They also run this platform on-premises and on other public clouds. Building a platform that targets multiple clouds is not an easy task. You have to find the right balance between going with full managed 1st party technologies (most of the time available as PaaS/SaaS, so easy to use / operate), and cross-platforms technologies coming from the [cloud-native landscape](https://landscape.cncf.io/). You have to find a way to abstract the underlying platform from your user, from your pipelines, but also make sure you do not take any architecture decision that could be influenced by how you designed something on another cloud or on-premise. In this article, I will share some learnings and go through different items that might be useful for you if you are working on such a project. I will try to answer some specific question like: How to ensure your application are agnostic from the underlying platform? How you ensure you have consistency between platforms in your monitoring and logging stack? How you ensure you can reuse the same tools and process in your DevOps pipelines? And most importantly, as I will focus on the Microsoft Azure platform: How you can get the best of Azure, without reinventing the wheel, when building a cloud native platform?

<!--more-->

## Cloud Native, what does it mean?

The Cloud Native Computing Foundation (CNCF) provides [one official definition](https://github.com/cncf/foundation/blob/master/charter.md) of what means "cloud native":

*Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.*

*These techniques enable loosely coupled systems that are resilient, manageable, and observable. Combined with robust automation, they allow engineers to make high-impact changes frequently and predictably with minimal toil.*

## Kubernetes as the core abstraction layer

No doubt that Kubernetes is the central piece of cloud-native platform these days, as it is designed to provide abstraction of the infrastructure on top of which it is running. Kubernetes comes with a complete deployment and development model to ensure that applications running in Kubernetes do not have to know or interact with the actual machines they are running on, or about network, persistent storage, security boundaries they are using.

The goal of this article is not to go into details about the application layer itself and how to deploy applications inside a Kubernetes cluster. There is already tons of articles and documentations about that. If you want to know more about how to get started with application development/deployment with Kubernetes on Azure, you can start from [there](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads).

What I am interested in sharing today is the hidden side of the iceberg: how it's possible to build on top of Azure Kubernetes Service (AKS) and provide to internal or external customers a platform where they can deploy applications, without even thinking or knowing that it actually runs on Azure. Actually, it's not only about AKS. It's about how it's possible to automate the provisioning of an enterprise-grade, high scale, secured Kubernetes on Azure.

### Landing Zone

All begins with a landing zone. This is the basic infrastructure where the platform will be deployed. It includes components such as Azure subscription, network topology and connectivity (express route, firewall, vnet, route tables, dns configuration...), identity and role-based access control, business continuity and disaster recovery.

Read more about Azure Landing Zone [here](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/), part of the Cloud Adoption Framework documentation.

Landing Zones are mostly about automation with intensive use of infrastructure as code (ARM template, Terraform, Bicep...) to make sure it's possible to create one quickly, in an automated and reproductible way. 

### Private Cluster

### High availability, High Scale

#### Support for availability zone
#### Cluster-Autoscaler

### Azure Active Directory Integration

https://docs.microsoft.com/en-us/azure/aks/managed-aad

### KeyVault Secret Management

https://docs.microsoft.com/en-us/azure/aks/csi-secrets-store-driver

## DevOps

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


---
layout: post
title:  "Terraform on Microsoft Azure - Part 1: Introduction"
date:   2019-09-04 10:20:00 +0200
categories: 
- Microsoft Azure
- DevOps
- Terraform
author: 'Julien Corioland'
identifier: '1465446e-1175-4bab-b4f1-c02eb42c59a2'
---

Recently, I have been involved in several projects to help customers to improve the way they are dealing with their infrastructure deployments. Cloud platforms like Microsoft Azure enable a high level of automation of these deployments and there are several options and tools available to you to make sure that you are able to deploy the different components of your infrastructure in an automated, repeatable and idempotent way.

In this blog post series I will detail some best practices about using Terraform and setting up continuous deployment and testing for Microsoft Azure infrastructure.

![Terraform on Azure](/images/terraform-microsoft-azure-introduction/terraform-azure.png)

<!--more-->

Before going deep dive into the technical details, I'd like to come back briefly on the concept of **Infrastructure as Code**, which is really important.

If it's relatively common for development teams to centralize the source code of an application into a source code repository (like GitHub or Azure Repos, for example) to be able to share it, to merge / resolve conflicts between developers working on the same items, to version it and eventualy to revert to any version at any time, it may not be the case for all infrastructure / IT teams.

But it is totally possible today to apply the exact same patterns and practices to your infrastructure, just by treating it as code. That means that all the scripts, configuration files and templates that you are used to write to describe, deploy and configure your infrastructure can also be stored in a source control repository, be versioned and have the exact same lifecycle than applications that are being deployed on this infrastructure.

This is really important because, by doing that, you will be able to revert at any time to any version of an application and redeploy it onto the right version of the infrastructure, without asking you how you (or the guy before you've join the company) were doing it at this time.

They key here is **automation**: by implementing Infrastructure as Code (IaC) you will be able to deliver infrastructure continuously and improving its quality by running automated tests on it.

There are a lot of tools out there that can help you in mastering infrastructure as code and continuous deployment practices for your infrastructure. You may already have used Azure Resource Manager templates, those JSON files that allow you to describe the whole infrastructure, with some variables and parameters and that you can use to deploy to Azure, sometime into different environments like development, tests, production…

[Terraform](https://www.terraform.io/) is an open source tool developed and maintained by [HashiCorp](https://www.hashicorp.com/) that has the exact same goal than ARM templates: it helps you to describe your infrastructure, using [HCL (HashiCorp Configuration Language)](https://www.terraform.io/docs/configuration/syntax.html) which is more readable than JSON, and then deploy it to Azure.
It comes with a set of tools that also help you to manage the state and lifecycle of your deployments, which makes it a bit more advanced than ARM templates.

Also, Terraform is not only working with Microsoft Azure, but also with a ton of other providers (the full list is available [here](https://www.terraform.io/docs/providers/)). That does not mean that when you write an HCL template for Microsoft Azure, then it can be used to deploy on any other cloud magically. No, that means that you will be able to use the same language and tools to automate the way you deploy a specific infrastructure on any platforms supported by Terraform. Trust me, it's something big!

In this blog post series, I will guide you into several topics I've gone through with the different devops teams I've worked with over the past months.

The goal will be to illustrate how you can use Terraform and Microsoft Azure to deploy the following infrastructure:

[![Microsoft Azure infrastructure overview](/images/terraform-microsoft-azure-introduction/azure-infrastructure-overview.jpg)](/images/terraform-microsoft-azure-introduction/azure-infrastructure-overview.jpg)
<center><i><small>Microsoft Azure infrastructure Overview - Click to zoom</small></i></center>

As you can see on the picture above, there are different kind of components that compose the infrastructure:

- core components like virtual networks and subnets
- infrastructure services like Azure Kubernetes Services or simple Azure virtual machines
- platform services, like Azure Database for MySql, KeyVault, Container Registry...
- common/transverse tools Azure Monitor and Log Analytics

I've also chosen to have components with different lifecycles, because they are duplicated environment that can be created and removed on demand (dev, QA, production...) or because they are common components that are used by other part of the infrastructure.

Those deliberated choices will help to discuss different topics like:
- how to factorize infrastructure code by writing Terraform modules?
- how to manage infrastructure deployment with components that have different lifecycles?
- how you deploy only a part of your infrastructure, like a duplicated environment or a subset of an environment?
- how you can rollback to a previous version of an application and its underlying infrastructure?

To answer all those questions, this blog post series will be composed of the following articles:

- Terraform on Microsoft Azure - Part 1: Introduction (this article)
- [Terraform on Microsoft Azure - Part 2: Basics](/archives/2019/09/04/terraform-microsoft-azure-basics.html)
- [Terraform on Microsoft Azure - Part 3: Terraform Remote State Management](/archives/2019/09/09/terraform-microsoft-azure-remote-state-management.html)
- Terraform on Microsoft Azure - Part 4: Writing Terraform modules
- Terraform on Microsoft Azure - Part 5: How to test your Terraform deployments?
- Terraform on Microsoft Azure - Part 6: Continuous integration using Azure Pipeline
- Terraform on Microsoft Azure - Part 7: Continuous deployment using Azure Pipeline

I hope this blog post series will help you to get started with infrastructure deployments using Terraform on Microsoft Azure or help you to improve what you already have done so far!

Looking forward to your feedbacks!

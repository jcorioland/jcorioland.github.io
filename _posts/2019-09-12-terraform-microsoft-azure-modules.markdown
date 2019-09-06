---
layout: post
title:  "Terraform on Microsoft Azure - Part 4: Terraform projects organization and modules"
date:   2019-09-12 10:00:00 +0200
categories: 
- Microsoft Azure
- DevOps
- Terraform
author: 'Julien Corioland'
identifier: '1465446e-1175-4bab-b4f1-c02eb42c59a4'
---

This blog post is part of the series about using [Terraform on Microsoft Azure](https://blog.jcorioland.io/archives/2019/09/04/terraform-microsoft-azure-introduction.html). In this part, I will discuss about how you can organize your Terraform files and how to maximize code reuse, especially using Terraform modules.

<!--more-->

Infrastructure as Code is about following the same practices with infrastructure deployment templates than with application code. One of the golden rule is to try to mutualize the code whenever it is possible to do it. 
Like developers, try to avoid copy/past portions of code from one file to another. It is not always an easy task, but there are some tools, like Terraform modules that will help you to achieve this goal.

## Terraform Project structure

Before going deep dive into Terraform modules, let's discuss about the basic structure/organization of a Terraform project.
You already know from [the second article of this blog posts series](https://blog.jcorioland.io/archives/2019/09/04/terraform-microsoft-azure-basics.html) that a Terraform project is a collection of `*.tf` files in a specific directory.

Here is what IMHO must look like a minimal Terraform project directory:

```
/myproject
-- main.tf
-- feature1.tf
-- feature2.tf
-- outputs.tf
-- variables.tf
-- README.md
```

If your Terraform code is mixed with application source code (which is great!), you can isolate it into a dedicated folder:

```
/myproject
-- /src
---- app stuff
-- /tf
---- main.tf
---- feature1.tf
---- feature2.tf
---- outputs.tf
---- variables.tf 
---- README.md
```

You can even go further and have a sub-directory per environment, if you want to manage configuration from your git repository or if your infrastructure is different dependening on the environment:

```
/myproject
-- /src
---- app stuff
-- /tf
---- /dev
------ main.tf
------ feature1.tf
------ feature2.tf
------ outputs.tf
------ variables.tf 
------ README.md
---- /prod
------ main.tf
------ feature1.tf
------ feature2.tf
------ feature3.tf
------ outputs.tf
------ variables.tf 
------ README.md
```

Let's have a look to the [`core`](https://github.com/jcorioland/terraform-azure-reference/tree/master/tf/core) configuration on the reference implementation that I am using as support for this blog post series (forgot about modules for now, we'll discuss this later in this post).

This configuration is reponsible for:
- creating the resource group, for a given environment (dev, test, production...)
- creating the virtual networks and subnets, for a given environment

*Note: the reference implementation architecture diagram is available [here](https://github.com/jcorioland/terraform-azure-reference/blob/master/assets/architecture.jpg), if you want to refresh your mind.*

- `main.tf`: technically speaking, this is not the entrypoint of the project (each files are loaded in alphabetical order) but this file usually contains provider configuration, backend configuration, imports to the modules to use and eventually some common resources that was not isolated into a specific file. For small projects, it's also OK to have the whole infrastructure configuration defined here. When it starts to get bigger, split it into other `*.tf` files.

```hcl
provider "azurerm" {
  version = "~> 1.31"
}

terraform {
  backend "azurerm" {}
}

resource "azurerm_resource_group" "rg" {
  name     = "tf-ref-${var.environment}-rg"
  location = "${var.location}"
}

resource "azurerm_virtual_network" "aks" {
  name                = "aks-vnet"
  address_space       = ["10.1.0.0/16"]
  location            = "${azurerm_resource_group.rg.location}"
  resource_group_name = "${azurerm_resource_group.rg.name}"
}

resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = "${azurerm_resource_group.rg.name}"
  virtual_network_name = "${azurerm_virtual_network.aks.name}"
  address_prefix       = "10.1.0.0/24"
}


resource "azurerm_virtual_network" "backend" {
  name                = "backend-vnet"
  address_space       = ["10.2.0.0/16"]
  location            = "${azurerm_resource_group.rg.location}"
  resource_group_name = "${azurerm_resource_group.rg.name}"
}

resource "azurerm_subnet" "backend" {
  name                 = "backend-subnet"
  resource_group_name  = "${azurerm_resource_group.rg.name}"
  virtual_network_name = "${azurerm_virtual_network.backend.name}"
  address_prefix       = "10.2.0.0/24"
}

resource "azurerm_virtual_network_peering" "peering1" {
  name                      = "aks2backend"
  resource_group_name       = "${azurerm_resource_group.rg.name}"
  virtual_network_name      = "${azurerm_virtual_network.aks.name}"
  remote_virtual_network_id = "${azurerm_virtual_network.backend.id}"
}

resource "azurerm_virtual_network_peering" "peering2" {
  name                      = "backend2aks"
  resource_group_name       = "${azurerm_resource_group.rg.name}"
  virtual_network_name      = "${azurerm_virtual_network.backend.name}"
  remote_virtual_network_id = "${azurerm_virtual_network.aks.id}"
}
```

- `outputs.tf`: contains the definitions for the deployment output variables, i.e. all the information that you want to retrieve and output the deployment

```hcl
output "resource_group_name" {
  value = "${azurerm_resource_group.rg.name}"
}

output "location" {
  value = "${var.location}"
}

output "environment" {
  value = "${var.environment}"
}
```

- `variables.tf`: contains the definitions of the variables that are used in this configuration projects.

```hcl
variable "environment" {
  description = "Name of the environment"
}

variable "location" {
  description = "Azure location to use"
}
```

- other `*.tf` files: contain the definition of the resources and data sources related to each other that you are using to deploy your infrastructure (for example `network.tf` and `virtual_machine.tf`...)
- `README.md`: it's always useful to document what the Terraform project does. A readme file is perfect for that :)

## Resource definition VS Data Source

There are two ways to reference an instance of a service running in Azure with Terraform. You can use a resource definition, with the `resource` keyword, like this is done in the snippets above or you can use a data source, with the `data` keyword:

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "tf-ref-${var.environment}-rg"
  location = "${var.location}"
}

data "azurerm_resource_group" "rg" {
  name = "tf-ref-${var.environment}-rg"
}
```

When you use the `resource` keyword, you indicate to Terraform that the current configuration is in charge of managing the lifecycle of the object, i.e. to create/update it when `terraform apply` is called or to destroy it when the `terraform destroy` command is called.

When you use the `data` keyword, you indicate to Terraform that you only want to get a reference of the existing object, but don't want to manage it part of this configuration (because it's managed by another team, another module etc...). If the object does not exist when you apply the configuration, the Terraform command will fail.

Once you have a resource or a data source reference, you can use it in other part of your template, using it's resource / data source name (in that case `rg` - the string that comes after the object type) like the following:

```hcl
# reference a resource
resource_group_name = "${azurerm_resource_group.rg.name}"

# reference a data source
resource_group_name = "${data.azurerm_resource_group.rg.name}"
```

Understanding the difference between those two types of object in really important to be able to write module and manage dependencies between all the modules that compose your solution!

## Writing a Terraform module

Now that you know about the basic structure of a Terraform configuration project and start to get familiar with the syntax, we can discuss about Terraform modules.

And the good news is that a module is nothing more than a directory with a bunch of Terraform files, structured like a project. Cool, isn't it? :)

Terraform modules are used to group together a set of resources that have the same lifecycle. It is not mandatory to use modules, but in some case it might be useful.

Like all mechanisms that allow to mutualize/factorize code, modules can also be dangerous: you don't want to have a big module that contains everything that you need to deploy and make all the resources strongly coupled together. This could lead to a monolith that will be really hard to maintain and to deploy.

Here are some questions that you can ask yourself for before writing a module:

- Do have all the resources involved the same lifecycle?
  - Will the resources be deployed all together all the time?
  - Will the resources be updated all together all the time?
  - Will the resources be destroyed all together all the time?
- Is there multiple resources involved? If there is just one, the module is probably useless
- From an architectural/functionnal perspective, does it makes sense to group all these resources together? (network, compute, storage etc...)
- Does any of the resource involved depend from a resource that is not in this module?

If the answer to these questions is no most of the time, then you probably don't need to write a module.

Sometime, instead of writing a big module, it can be useful to write multiple ones and nest them together, depending on the scenario you want to cover.

You can write multiple modules into separate directory of your project, or you can write modules in separate repositories. It's also possible to import existing modules from the [Terraform Registry](https://registry.terraform.io/browse/modules?provider=azurerm).

In the reference implementation I am using for this blog post series, I have the [core module defined in the main repository](https://github.com/jcorioland/terraform-azure-reference/tree/master/tf/core) on other modules like the Azure Kubernetes Service one, defined in its [own GitHub repository](https://github.com/jcorioland/terraform-azure-ref-aks-module).

If you compare the structure of both module, you'll see that it's exactly the same! The only thing that is going to change is how you are going to import it and use it into the main configuration project. We will discuss about that in the next section.

Look into the AKS module `main.tf` [file](https://github.com/jcorioland/terraform-azure-ref-aks-module/blob/40a2801aa1b28c54d9ab8f078677784fd00a2548/main.tf#L9). 

```hcl
data "azurerm_resource_group" "rg" {
  name = "tf-ref-${var.environment}-rg"
}

data "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  virtual_network_name = "aks-vnet"
  resource_group_name  = "${data.azurerm_resource_group.rg.name}"
}
```

You can see that I've used the `data` keyword to reference the resource group and the subnet where AKS needs to be deployed. In that case, this is because I made the assumption that those two resources that are part of the core module need to be deployed before AKS. In other words, I have created a dependency between the two modules. That allows me to manage their lifecycle differently. Or it could even be managed by two different teams in the company, if needed.

## Using a Terraform module

Like any Terraform configuration project, a Terraform module takes input parameters (variables), creates some pieces of infrastructure and return output values.

To import a Terraform module into another project, use the `module` directive:

```hcl
module "tf-ref-aks-module" {
  source                           = "../../"
  environment                      = "Development"
  location                         = "francecentral"
  kubernetes_version               = "1.14.6"
  service_principal_client_id      = "CLIENT_ID"
  service_principal_client_secret  = "CLIENT_SECRET"
}
```

Depending on where the module is located (in a sub-directory, in another GitHub repo, in the Terraform registry...) and from where it is used, the `source` parameter may be different:

```hcl
provider "azurerm" {
  version = "~>1.30"
}

terraform {
  backend "azurerm" {}
}

module "aks" {
  source                          = "git@github.com:jcorioland/terraform-azure-ref-aks-module"
  environment                     = "${var.environment}"
  location                        = "${var.location}"
  kubernetes_version              = "${var.kubernetes_version}"
  service_principal_client_id     = "${var.service_principal_client_id}"
  service_principal_client_secret = "${var.service_principal_client_secret}"
  ssh_public_key                  = "${var.ssh_public_key}"
}
```

Once you've created your configuration that aggregates all the modules you want to import, just call the `terraform apply` command. It is simple as that!

You can read more about Terraform modules on [this page of the Terraform documentation](https://www.terraform.io/docs/modules/index.html).

## Conclusion

In this blob post I gave you some tips and tricks on how you can organize your Terraform projects and use modules to maximize code reuse in your projects.

As explained, there ar a lot of options to achieve these goals and the choices done will be different depending on the project you are working on. That's being said, I hope you've enjoyed the read and that it gave you enough insights to start thinking about your own Terraform projects organization!

In the next post of this series, I will discuss about testing Terraform modules... Stay tuned!
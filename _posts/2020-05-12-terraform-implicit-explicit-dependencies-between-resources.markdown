---
layout: post
title:  "How to manage implicit and explicit dependencies with Terraform?"
date:   2020-05-12 10:00:00 +0200
categories: 
- Microsoft Azure
- Terraform
author: 'Julien Corioland'
identifier: '9b078f9b-a1f7-4e37-b68f-ec086d9935a6'
image: /images/terraform-implicit-explicit-dependencies-between-resources/graph-with-depends-on.jpg
---

Recentely, I and my colleague [April](https://twitter.com/TheAprilEdwards) have been struggling on an error that was happening randomly when calling `terraform destroy` on a Terraform module we were working on. You know, this kind of issue that first gives you a lot of satisfaction when you solve it, but then frustration because the solution is so simple that you are wondering why you spent so much time on it! Everything was about explicit and implicit dependencies between Terraform resources. Let me explain...

<!--more-->

## Some context

Before detailling the issue, let me give you some context. The Terraform module *(Module B, on the diagram below)* we were working on is responsible for deploying resources (virtual machines, application security group (ASG) etc.) into an existing virtual network, managed by another Terraform module *(Module A, on the diagram below)*, and adding some network security rules related to the ASG into a network security group (NSG) also managed by the other Terraform module *(Module A, on the diagram below)*. Looking something like the following:

![Architecture Diagram](/images/terraform-implicit-explicit-dependencies-between-resources/architecture-diagram.jpg)

And now, the error we were encountering, *sometime*, when calling `terraform destroy`:

```console
Error waiting for removal of Application Security Group for NIC "XXX" (Resource Group "rg-XXX"): Code="OperationNotAllowed" Message="Operation 'startTenantUpdate' is not allowed on VM 'XXX' since the VM is marked for deletion. You can only retry the Delete operation (or wait for an ongoing one to complete).
```

OK, seems like there is something wrong, *sometime*, in the way Terraform asks to Microsoft Azure platform to delete the different resources...

## The Terraform code

First, let's look at the Terraform code to understand the relationship between network interface card (NIC), ASG and the virtual machine, as these are the three resources involved in the error.

```hcl
data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

data "azurerm_subnet" "subnet" {
  name                 = var.subnet_name
  virtual_network_name = var.vnet_name
  resource_group_name  = var.vnet_resource_group_name
}

data "azurerm_application_security_group" "asg" {
  name                = var.asg_name
  resource_group_name = data.azurerm_resource_group.rg.name
}

resource "azurerm_network_interface" "nic" {
  name                = "nic-${var.suffix}"
  location            = data.azurerm_resource_group.rg.location
  resource_group_name = data.azurerm_resource_group.rg.name

  ip_configuration {
    subnet_id                      = data.azurerm_subnet.subnet.id
    application_security_group_ids = [data.azurerm_application_security_group.asg.id]
  }
}

resource "azurerm_network_interface_application_security_group_association" "vm_asg_assoc" {
  ip_configuration_name         = azurerm_network_interface.nic.ip_configuration[0].name
  network_interface_id          = azurerm_network_interface.nic.id
  application_security_group_id = data.azurerm_application_security_group.asg.id
}

resource "azurerm_virtual_machine" "vm" {
  
  network_interface_ids = [azurerm_network_interface.nic.id]

}
```

> Note: This is an extract of the module that deploys a virtual machine, link it to the existing virtual network and to the application security group. It is not a working code sample as I have removed all parts/properties that are not related to NIC, VNET/Subnet and VMs, for readibility.

## Implicit vs Explicit dependencies in Terraform

Terraform deals with two kinds of dependencies between the resources it manages: implicit dependencies and explicit dependencies. Implicit dependencies, like their names suggest, are automatically detected by Terraform. For example, in the code below, there is an implicit dependency between the network interface and the virtual machine, because the VM resource uses the network interface `id`:

```hlc
resource "azurerm_virtual_machine" "vm" {
  
  network_interface_ids = [azurerm_network_interface.nic.id]

}
```

It helps Terraform to know in what order the resources must be created and deleted. More specifically in this case, it means that the network interface must be created before the virtual machine and the virtual machine must be deleted before the network interface.

Explicit dependencies are dependencies that are set up "manually" between resources, using the `depends_on` keyword. **Spoiler alert**: this is what was missing in our code, and what is missing in the sample above, I'll come back on this later.

## Understand the Terraform graph

In some situation (like this one!), it is super useful to visualize the dependency graph of the different resources that compose a Terraform project / module. This is where the `terraform graph` command comes to the rescue:

```bash
jcorioland@devbox-julien:~/sources/nsg-asg-vm-destroy-issue-repro/src/environment-module/vm-submodule$ terraform graph
digraph {
  compound = "true"
  newrank = "true"
  subgraph "root" {
    "[root] azurerm_network_interface.nic" [label = "azurerm_network_interface.nic", shape = "box"]
    "[root] azurerm_network_interface_application_security_group_association.vm_asg_assoc" [label = "azurerm_network_interface_application_security_group_association.vm_asg_assoc", shape = "box"]
    "[root] azurerm_virtual_machine.vm" [label = "azurerm_virtual_machine.vm", shape = "box"]
    "[root] data.azurerm_application_security_group.asg" [label = "data.azurerm_application_security_group.asg", shape = "box"]
    "[root] data.azurerm_resource_group.rg" [label = "data.azurerm_resource_group.rg", shape = "box"]
    "[root] data.azurerm_subnet.subnet" [label = "data.azurerm_subnet.subnet", shape = "box"]
    "[root] output.ssh_private_key" [label = "output.ssh_private_key", shape = "note"]
    "[root] output.ssh_public_key" [label = "output.ssh_public_key", shape = "note"]
    "[root] output.vm_name" [label = "output.vm_name", shape = "note"]
    "[root] provider.azurerm" [label = "provider.azurerm", shape = "diamond"]
    "[root] provider.tls" [label = "provider.tls", shape = "diamond"]
    "[root] tls_private_key.ssh" [label = "tls_private_key.ssh", shape = "box"]
    "[root] var.asg_name" [label = "var.asg_name", shape = "note"]
    "[root] var.nsg_name" [label = "var.nsg_name", shape = "note"]
    "[root] var.resource_group_name" [label = "var.resource_group_name", shape = "note"]
    "[root] var.subnet_name" [label = "var.subnet_name", shape = "note"]
    "[root] var.suffix" [label = "var.suffix", shape = "note"]
    "[root] var.vnet_name" [label = "var.vnet_name", shape = "note"]
    "[root] var.vnet_resource_group_name" [label = "var.vnet_resource_group_name", shape = "note"]
    "[root] azurerm_network_interface.nic" -> "[root] data.azurerm_application_security_group.asg"
    "[root] azurerm_network_interface.nic" -> "[root] data.azurerm_subnet.subnet"
    "[root] azurerm_network_interface.nic" -> "[root] var.suffix"
    "[root] azurerm_network_interface_application_security_group_association.vm_asg_assoc" -> "[root] azurerm_network_interface.nic"
    "[root] azurerm_virtual_machine.vm" -> "[root] azurerm_network_interface.nic"
    "[root] azurerm_virtual_machine.vm" -> "[root] tls_private_key.ssh"
    "[root] data.azurerm_application_security_group.asg" -> "[root] data.azurerm_resource_group.rg"
    "[root] data.azurerm_application_security_group.asg" -> "[root] var.asg_name"
    "[root] data.azurerm_resource_group.rg" -> "[root] provider.azurerm"
    "[root] data.azurerm_resource_group.rg" -> "[root] var.resource_group_name"
    "[root] data.azurerm_subnet.subnet" -> "[root] provider.azurerm"
    "[root] data.azurerm_subnet.subnet" -> "[root] var.subnet_name"
    "[root] data.azurerm_subnet.subnet" -> "[root] var.vnet_name"
    "[root] data.azurerm_subnet.subnet" -> "[root] var.vnet_resource_group_name"
    "[root] meta.count-boundary (EachMode fixup)" -> "[root] azurerm_network_interface_application_security_group_association.vm_asg_assoc"
    "[root] meta.count-boundary (EachMode fixup)" -> "[root] output.ssh_private_key"
    "[root] meta.count-boundary (EachMode fixup)" -> "[root] output.ssh_public_key"
    "[root] meta.count-boundary (EachMode fixup)" -> "[root] output.vm_name"
    "[root] meta.count-boundary (EachMode fixup)" -> "[root] var.nsg_name"
    "[root] output.ssh_private_key" -> "[root] tls_private_key.ssh"
    "[root] output.ssh_public_key" -> "[root] tls_private_key.ssh"
    "[root] output.vm_name" -> "[root] azurerm_virtual_machine.vm"
    "[root] provider.azurerm (close)" -> "[root] azurerm_network_interface_application_security_group_association.vm_asg_assoc"
    "[root] provider.azurerm (close)" -> "[root] azurerm_virtual_machine.vm"
    "[root] provider.tls (close)" -> "[root] tls_private_key.ssh"
    "[root] root" -> "[root] meta.count-boundary (EachMode fixup)"
    "[root] root" -> "[root] provider.azurerm (close)"
    "[root] root" -> "[root] provider.tls (close)"
    "[root] tls_private_key.ssh" -> "[root] provider.tls"
  }
}
```

Even better, you can use the following command to export it as SVG graph:

```bash
terraform graph | dot -Tsvg > graph.svg
```

And voilà, a beautiful dependency graph that really helps to figure out what's going on:

![First Dependency Graph - no explicit dependency](/images/terraform-implicit-explicit-dependencies-between-resources/graph-no-depends-on.jpg)

Looking at the three resources that are important for us in that case, we can clearly see that there is no implicit dependencies between the virtual machines and the application security group. They just share a common dependency, the network interface, but not directly, through the association between the ASG and the NIC (the `azure_network_interface_application_security_group_association` resource).

## Solving the issue

Having the dependency graph in mind, it appears clearly that to fix the issue, we must have an explicit dependency between the virtual machine and the association between the ASG and the NIC. This is where the `depends_on` keyword will be helpful. We can make sure that the association is always created before the virtual machine is, and that the virtual machine is always deleted before the association is, by adding a `depends_on` block on the virtual machine resources:

```hcl
resource "azurerm_virtual_machine" "vm" {
  
  network_interface_ids = [azurerm_network_interface.nic.id]

  depends_on = [
    azurerm_network_interface_application_security_group_association.vm_asg_assoc
  ]

}
```

Let's try to regenerate the graph again:

![Second Dependency Graph - with explicit dependency](/images/terraform-implicit-explicit-dependencies-between-resources/graph-with-depends-on.jpg)

As you can see on the graph, now the dependency is explicit, and it's easy for Terraform to know that it must delete the association between NIC and ASG before deleting the virtual machine resources.

Bug fixed! :-)

## Conclusion

I hope this real use case will help you to understand better the dependency graph that Terraform builds to manage dependencies between the different resources to apply and destroy them in order, but also how to deal with explicit and implicit dependencies and sometime go deeper into the graph to understand a situation that is not obvious at first look...

You can also read April's point of view on this issue [here](https://azapril.dev/2020/05/12/terraform-depends_on/).

Cheers!

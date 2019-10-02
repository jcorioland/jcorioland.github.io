---
layout: post
title:  "Terraform on Microsoft Azure - Part 7: Continuous Deployment using Azure Pipeline"
date:   2019-10-02 10:00:00 +0200
categories: 
- Microsoft Azure
- DevOps
- Terraform
author: 'Julien Corioland'
identifier: '1465446e-1175-4bab-b4f1-c02eb42c59a7'
image: /images/terraform-microsoft-azure-introduction/terraform-azure.png
---

This blog post is part of the series about using [Terraform on Microsoft Azure](https://blog.jcorioland.io/archives/2019/09/04/terraform-microsoft-azure-introduction.html). In the [previous article](https://blog.jcorioland.io/archives/2019/09/25/terraform-microsoft-azure-ci-docker-azure-pipeline.html), I explained how to use Docker and Azure Pipeline to continuously integrate and tests Terraform infrastructure modules / deployments. In this new blog post, I will discuss about using Azure Pipeline to actually deploy the infrastructure continuously.

<!--more-->

*Note: this blog post series comes with a [reference implementation](https://github.com/jcorioland/terraform-azure-reference) hosted on my GitHub. Do not hesitate to check it out to go deeper into the details, fork it, contribute, open issues... :)*

So far, I've discussed about the different modules that compose the reference architecture independently one from each other. The idea of this post is to explain how we can create a deployment pipeline that will deploy the whole infrastructure. 

## Azure multi-stages pipelines

To achieve this goal, I have used the new YAML Azure multi-stages pipeline:

![Azure Multi-Stages Pipeline](/images/terraform-microsoft-azure-pipeline/multi-stages-pipeline.png)

Azure multi-stages pipelines allow to specify the whole deployment pipeline using a YAML file that lives inside the source code repository and evolve with your infrastructure. This is sometime referenced as *pipeline as code*. If you are not familiar with Azure DevOps YAML pipeline, I strongly recommend that you [give it a try](https://docs.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline?view=azure-devops&tabs=tfs-2018-2) before continuing to read this article. It's really awesome!

*Note: at the time I am writing this article, multi-stages pipelines are still in preview and need to be activated on your Azure DevOps account / organization. Check out [this documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/stages?view=azure-devops&tabs=yaml) for more information.*

The [main repository](https://github.com/jcorioland/terraform-azure-reference/tree/master/tf) of the Terraform on Azure reference implementation contains the definition of what we want to deploy using Azure Pipeline. Each directory contains the Terraform code used to deploy the three modules: common, core networking and Azure Kubernetes Service.

I've chosen to deploy each module into a separate stage. In Azure Pipeline, a stage is a way to organize the different jobs and tasks that are going to be executed. One advantage of splitting into different stages is that you can use conditions to execute or not a given stage, depending on what you want to deploy. For example, you can have a condition that will skip some deployment steps if they are already done.

## Deploy a Terraform module as a pipeline stage

Because we are using Terraform, we are going to repeat the exact same steps to deploy any module to Azure:
- Install an SSH Key on the pipeline agent (used to connect to GitHub, optional dependending on how you deploy/reference your modules)
- Install Terraform on the agent
- Run Terraform init
- Run Terraform validate
- Run Terraform apply

Each step is identified as a task in the YAML pipeline:

```yaml
- stage: CommonModule
    displayName: Common Module
    jobs:
    # Common Module
    - job: CommonModule
      displayName: Deploy the Terraform Common module
      pool:
        vmImage: $(vmImageName)
      steps:
      - task: InstallSSHKey@0
        displayName: 'Install an SSH key'
        inputs:
          knownHostsEntry: $(sshKnownHostsEntry)
          sshPublicKey: $(sshPublicKey)
          sshKeySecureFile: $(sshKeySecureFileName)
      - task: charleszipp.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-installer.TerraformInstaller@0
        displayName: 'Use Terraform $(terraformVersion)'
        inputs:
          terraformVersion: $(terraformVersion)
      - task: charleszipp.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@0
        displayName: 'terraform init'
        inputs:
          command: init
          workingDirectory: '$(System.DefaultWorkingDirectory)/tf/common'
          backendType: azurerm
          backendServiceArm: $(azureSubscriptionServiceConnectionName)
          ensureBackend: true
          backendAzureRmResourceGroupLocation: $(location)
          backendAzureRmResourceGroupName: $(tfStateResourceGroupName)
          backendAzureRmStorageAccountName: $(tfStateAzureStorageAccountName)
          backendAzureRmStorageAccountSku: $(tfStateAzureStorageAccountSku)
          backendAzureRmContainerName: $(tfStateContainerName)
          backendAzureRmKey: 'common.tfstate'
      - task: charleszipp.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@0
        displayName: 'terraform validate'
        inputs:
          workingDirectory: '$(System.DefaultWorkingDirectory)/tf/common'
      - task: charleszipp.azure-pipelines-tasks-terraform.azure-pipelines-tasks-terraform-cli.TerraformCLI@0
        displayName: 'terraform apply'
        inputs:
          command: apply
          workingDirectory: '$(System.DefaultWorkingDirectory)/tf/common'
          environmentServiceName: $(azureSubscriptionServiceConnectionName)
          commandOptions: '-auto-approve -var location=$(location) -var tenant_id=$(tenantId)'
```

*Note: I have used [this Azure DevOps task from the marketplace](https://marketplace.visualstudio.com/items?itemName=charleszipp.azure-pipelines-tasks-terraform) to be able to work with Terraform inside the pipeline. You need to install it on your Azure DevOps organization, as a prerequisite.*

This task allows to manage [Terraform remote state management](https://blog.jcorioland.io/archives/2019/09/09/terraform-microsoft-azure-remote-state-management.html) inside the pipeline, by specifying the information about the Azure storage account to use to store the state:

```yaml
backendAzureRmResourceGroupLocation: $(location)
backendAzureRmResourceGroupName: $(tfStateResourceGroupName)
backendAzureRmStorageAccountName: $(tfStateAzureStorageAccountName)
backendAzureRmStorageAccountSku: $(tfStateAzureStorageAccountSku)
backendAzureRmContainerName: $(tfStateContainerName)
backendAzureRmKey: 'common.tfstate'
```

The complete Azure Pipeline is available [here](https://github.com/jcorioland/terraform-azure-reference/blob/master/azure-pipelines.yml).

## Azure DevOps pipelines and variables

Azure DevOps allows to use different kind of variables inside a pipeline, by referencing them using the $(VARIABLE_NAME) notation.

It can be:

- a variable defined inline in the YAML pipeline:

```yaml
variables:
  vmImageName: 'ubuntu-latest'
  terraformVersion: 0.12.3
  azureSubscriptionServiceConnectionName: 'jucoriol'
  tfStateResourceGroupName: 'terraform-ref-fr-rg'
  tfStateAzureStorageAccountSku: 'Standard_LRS'
  tfStateAzureStorageAccountName: 'tfstate201910'
  tfStateContainerName: 'tfstate-ref'
```

- a variable (or secret variable) defined through the portal, in the pipeline editor (once you've created it):

![Azure Pipeline Variables](/images/terraform-microsoft-azure-pipeline/pipeline-variables.png)

*Note: for more information about variables, check [this page](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch) of the documentation.*

## Triggering the pipeline

The pipeline also contains the definition for the trigger that will (or not) deploy the infrastructure as soon as a new commit is done on a given branch. In that case, I've configured it to be trigger manually (i.e. to not be triggered :-)):

```yaml
trigger: none
```

*Note: check [this page of the documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers?view=azure-devops&tabs=yaml) to learn more about Azure Pipelines triggers.*

## Running the pipeline

Once the YAML pipeline has been defined, you can run it from the Azure DevOps portal or from a trigger.

![Run Azure Pipeline](/images/terraform-microsoft-azure-pipeline/run-deployment.png)

That's all! Having your pipeline defined in YAML right inside your code repository will make really easy to make the infrastructure evolving continuously.

## Conclusion

In this blog post, I explained how to implement continuous infrastructure deployment using Terraform and Azure Pipeline. It concludes (for now :-)) this series about using Terraform on Microsoft Azure. I tried to explain what best practices you can implement in term of [Terraform source code organization and modularization](https://blog.jcorioland.io/archives/2019/09/11/terraform-microsoft-azure-modules.html), [remote state management](https://blog.jcorioland.io/archives/2019/09/09/terraform-microsoft-azure-remote-state-management.html), [continuous integration](https://blog.jcorioland.io/archives/2019/09/25/terraform-microsoft-azure-ci-docker-azure-pipeline.html) and [testing](https://blog.jcorioland.io/archives/2019/09/18/terraform-microsoft-azure-how-to-test-deployment.html) and finally in this post, continuous deployment.

I really hope that you have enjoyed read it and that it will help you to implement infrastructure as code best practices in your projects!

![Terraform on Microsoft Azure](/images/terraform-microsoft-azure-introduction/terraform-azure.png)

Cheers!
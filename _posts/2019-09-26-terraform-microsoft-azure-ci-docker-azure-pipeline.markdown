---
layout: post
title:  "Terraform on Microsoft Azure - Part 6: Continuous Integration using Docker and Azure Pipeline"
date:   2019-09-26 10:00:00 +0200
categories: 
- Microsoft Azure
- DevOps
- Terraform
author: 'Julien Corioland'
identifier: '1465446e-1175-4bab-b4f1-c02eb42c59a6'
---

This blog post is part of the series about using [Terraform on Microsoft Azure](https://blog.jcorioland.io/archives/2019/09/04/terraform-microsoft-azure-introduction.html). In the [previous article](https://blog.jcorioland.io/archives/2019/09/18/terraform-microsoft-azure-how-to-test-deployment.html), I detailled how you can use the Terratest framework to create and run Golang integration tests for your Terraform deployment. In this new part, I will discuss about automating these tests using Docker containers and Azure Pipeline.

<!--more-->

*Note: this blog post series comes with a [reference implementation](https://github.com/jcorioland/terraform-azure-reference) hosted on my GitHub. Do not hesitate to check it out to go deeper into the details, fork it, contribute, open issues... :)*

Having continuous integration for Terraform code allows to make sure that your infrastructure stay valid every time you update the code. It is really easy to implement, especially using Docker containers and Azure Pipeline. With these tools, you will be able to make sure that each time you commit a piece of Terraform code inside your repository, a new build pipeline is triggered and your infrastructure is deployed and tested on a test environment.

## The Docker base image

In order to run the tests inside a Docker container, I have created a base Docker image available [here](https://hub.docker.com/r/jcorioland/azure-terratest). You can easily build it yourself from the following [Dockerfile](https://github.com/jcorioland/azure-terratest-docker/blob/master/Dockerfile):

```Dockerfile
FROM golang:1.12.6

# Define environment variables
ARG BUILD_TERRAFORM_VERSION="0.12.3"
ARG BUILD_TERRAFORM_OS_ARCH=linux_amd64
ARG BUILD_TERRATEST_LOG_PARSER_VERSION="v0.17.5"

ENV TERRAFORM_VERSION=${BUILD_TERRAFORM_VERSION}
ENV TERRAFORM_OS_ARCH=${BUILD_TERRAFORM_OS_ARCH}
ENV TERRATEST_LOG_PARSER_VERSION=${BUILD_TERRATEST_LOG_PARSER_VERSION}

# Update & Install tool
RUN apt-get update && \
    apt-get install -y build-essential unzip

# Install dep.
ENV GOPATH /go
ENV PATH /usr/local/go/bin:$GOPATH/bin:$PATH
RUN /bin/bash -c "curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh"

# Install Terraform
RUN curl -Os https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_${TERRAFORM_OS_ARCH}.zip && \
    curl -Os https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_SHA256SUMS && \
    curl -s https://keybase.io/hashicorp/pgp_keys.asc | gpg --import && \
    curl -Os https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_SHA256SUMS.sig && \
    gpg --verify terraform_${TERRAFORM_VERSION}_SHA256SUMS.sig terraform_${TERRAFORM_VERSION}_SHA256SUMS && \
    shasum -a 256 -c terraform_${TERRAFORM_VERSION}_SHA256SUMS 2>&1 | grep "${TERRAFORM_VERSION}_${TERRAFORM_OS_ARCH}.zip:\sOK" && \
    unzip -o terraform_${TERRAFORM_VERSION}_${TERRAFORM_OS_ARCH}.zip -d /usr/local/bin

# Cleanup
RUN rm terraform_${TERRAFORM_VERSION}_${TERRAFORM_OS_ARCH}.zip
RUN rm terraform_${TERRAFORM_VERSION}_SHA256SUMS
RUN rm terraform_${TERRAFORM_VERSION}_SHA256SUMS.sig

# Install Terratest Log Parser
RUN curl -OLs https://github.com/gruntwork-io/terratest/releases/download/${TERRATEST_LOG_PARSER_VERSION}/terratest_log_parser_${TERRAFORM_OS_ARCH} && \
    chmod +x terratest_log_parser_${TERRAFORM_OS_ARCH} && \
    mv terratest_log_parser_${TERRAFORM_OS_ARCH} /usr/local/bin/terratest_log_parser
```

All what is done in this Dockerfile is to install the required tools to run Terratest inside a container. It is almost the same that what you've done on your machine if you've read my previous article. 

*Note: I unfortunately cannot commit to maintain this Docker base image, so you may want to build your own one :) If you don't care, you can use it using the following Docker repository/tag: `jcorioland/azure-terratest:0.12.3`.*

## Running the test inside a Docker container

Now that we have a base image, we can execute the test inside a Docker container. You can look at the Azure Kubernetes Service Terraform module repository that is part of the reference implementation that comes with this series, and especially to the [Dockerfile](https://github.com/jcorioland/terraform-azure-ref-aks-module/blob/master/Dockerfile) and the [run-tests.sh](https://github.com/jcorioland/terraform-azure-ref-aks-module/blob/master/run-tests.sh):

### Dockerfile

```Dockerfile
FROM jcorioland/azure-terratest:0.12.3

ARG BUILD_MODULE_NAME="aks-module"
ENV MODULE_NAME=${BUILD_MODULE_NAME}

RUN ssh-keygen -b 2048 -t rsa -f ~/.ssh/testing_rsa -q -N ""

# Set work directory.
RUN mkdir /go/src/${MODULE_NAME}
COPY . /go/src/${MODULE_NAME}
WORKDIR /go/src/${MODULE_NAME}

RUN chmod +x run-tests.sh

ENTRYPOINT [ "./run-tests.sh" ]
```

This Dockerfile uses the base image built previously and copy the module sources inside the container. It also creates an SSH key that is used in the tests (to deploy AKS). Finally, it set the `run-tests.sh` script as an entrypoint.

### run-tests.sh

```bash
#!/bin/bash

set -e

# ensure dependencies
dep ensure

# set environment variables
export TF_VAR_service_principal_client_id=$SERVICE_PRINCIPAL_CLIENT_ID
export TF_VAR_service_principal_client_secret=$SERVICE_PRINCIPAL_CLIENT_SECRET

# run test
go test -v ./test/ -timeout 30m | tee test_output.log
terratest_log_parser -testlog test_output.log -outputdir test_output
```

This script runs the tests using the Golang test framework and parses the output logs.

To be able to run the tests, you need to build the Docker image, using:

```bash
docker built -t aks-module-tests ./
```

Now that the Docker image is built, you can basically run the tests inside a container:

```bash
# create an ouput directory on the machine to retrieve the testl results
mkdir ./test_output

# run the tests in the container
docker run \
	-e SERVICE_PRINCIPAL_CLIENT_ID=$(SERVICE_PRINCIPAL_CLIENT_ID) \
	-e SERVICE_PRINCIPAL_CLIENT_SECRET=$(SERVICE_PRINCIPAL_CLIENT_SECRET) \
	-e ARM_SUBSCRIPTION_ID=$(ARM_SUBSCRIPTION_ID) \
	-e ARM_CLIENT_ID=$(ARM_CLIENT_ID) \
	-e ARM_CLIENT_SECRET=$(ARM_CLIENT_SECRET) \
	-e ARM_TENANT_ID=$(ARM_TENANT_ID) \
	-v $(System.DefaultWorkingDirectory)/test_output:/go/src/aks-module/test_output \
	$(containerRegistry)/$(imageRepository):$(tag)
```

Some observations:

* a volume mount is created to be able to access the test results outside of the container, using the `-v` option
* some environment variables used by Terraform are passed to the container
  * `SERVICE_PRINCIPAL_CLIENT_ID`: service principal identifier used by AKS
  * `SERVICE_PRINCIPAL_CLIENT_SECRET`: service principal secret used by AKS
  * `ARM_SUBSCRIPTION_ID`: the id of the Azure subscription
  * `ARM_CLIENT_ID`: the client id to use to connect the Azure subscription
  * `ARM_CLIENT_SECRET`: the secret to use to connect the Azure subscription
  * `ARM_TENANT_ID`: the identifier of the Azure tenant

### Integrate with Azure Pipeline

The last step is to run the tests part of an Azure Pipeline. Now that we have a Docker image, it is really easy. The AKS module contains an [azure-pipeline.yaml](https://github.com/jcorioland/terraform-azure-ref-aks-module/blob/master/azure-pipelines.yml) file that defines all the steps (the same as above, actually):

```yaml
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- master

resources:
  containers:
  - container: 'aks_module_test'
    image: terraform-azure-reference/aks-module-tests:$(tag)
    endpoint: jcorioland_acr  

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: 'a0d4d283-d17d-4c9f-9154-c5c3be4f24c5'
  imageRepository: 'terraform-azure-reference/aks-module-tests'
  containerRegistry: 'jcorioland.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'
  
  # Agent VM image name
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: Build Container Image and Run Tests
  jobs:  
  - job: Build
    displayName: Build Container Image and Run Tests
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build Tests Container Image
      inputs:
        command: build
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
    - script: |
        mkdir ./test_output
        docker run \
          -e SERVICE_PRINCIPAL_CLIENT_ID=$(SERVICE_PRINCIPAL_CLIENT_ID) \
          -e SERVICE_PRINCIPAL_CLIENT_SECRET=$(SERVICE_PRINCIPAL_CLIENT_SECRET) \
          -e ARM_SUBSCRIPTION_ID=$(ARM_SUBSCRIPTION_ID) \
          -e ARM_CLIENT_ID=$(ARM_CLIENT_ID) \
          -e ARM_CLIENT_SECRET=$(ARM_CLIENT_SECRET) \
          -e ARM_TENANT_ID=$(ARM_TENANT_ID) \
          -v $(System.DefaultWorkingDirectory)/test_output:/go/src/aks-module/test_output \
          $(containerRegistry)/$(imageRepository):$(tag)
      displayName: Run tests in container
      env:
        SERVICE_PRINCIPAL_CLIENT_ID: $(SERVICE_PRINCIPAL_CLIENT_ID)
        SERVICE_PRINCIPAL_CLIENT_SECRET: $(SERVICE_PRINCIPAL_CLIENT_SECRET)
        ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
        ARM_CLIENT_ID: $(ARM_CLIENT_ID)
        ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
        ARM_TENANT_ID: $(ARM_TENANT_ID)
    - task: PublishTestResults@2
      inputs:
        testResultsFormat: 'JUnit' # Options: JUnit, NUnit, VSTest, xUnit, cTest
        testResultsFiles: '**/report.xml'
        failTaskOnFailedTests: true
```

*Note: check [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/docker?view=azure-devops) for more information about building docker image with Azure DevOps.*

This Azure Pipeline defines 1 jobs with 3 tasks:

* `Docker@2`: build the Docker image that will be used to execute the tests
* `script`: run the tests
* `PublishTestResults@2`: send the test results to Azure DevOps

It also indicates that each time a commit is done on the `master` branch, a pipeline is ran:

[![Azure Pipeline running the tests](/images/terraform-microsoft-azure-pipeline/run-pipeline.png)](/images/terraform-microsoft-azure-pipeline/run-pipeline.png)
<center><i><small>Azure Pipeline running tests in Docker container</small></i></center>

Once completed, you can see the test results directly in Azure DevOps dashboard:

[![Azure Pipeline test results](/images/terraform-microsoft-azure-pipeline/test-results.png)](/images/terraform-microsoft-azure-pipeline/test-results.png)
<center><i><small>Azure Pipeline Test results</small></i></center>

## Conclusion

In this blog post, I have explained how to use Terratest, Docker and Azure Pipeline to trigger an integration test as soon as new Terraform code is committed. Doing this will help you to maintain high quality infrastructure code. In the next post of the series, I will discuss about going to the next level and implement continuous deployment of the infrastructure using Azure Pipeline.

Stay tuned!
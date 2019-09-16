---
layout: post
title:  "Terraform on Microsoft Azure - Part 5: How to test your Terraform deployments?"
date:   2019-09-18 14:00:00 +0200
categories: 
- Microsoft Azure
- DevOps
- Terraform
author: 'Julien Corioland'
identifier: '1465446e-1175-4bab-b4f1-c02eb42c59a5'
---

This blog post is part of the series about using [Terraform on Microsoft Azure](https://blog.jcorioland.io/archives/2019/09/04/terraform-microsoft-azure-introduction.html). So far, I've discussed about Infrastructure as Code concepts, Terraform basics and best practices in term of remote state management, code organization and modules. In this new part, I'd like to give you some insights about how you can test your Terraform deployments.

<!--more-->

*Note: this blog post series comes with a [reference implementation](https://github.com/jcorioland/terraform-azure-reference) hosted on my GitHub. Do not hesitate to check it out to go deeper into the details, fork it, contribute, open issues... :)*

When it comes to automation, testing is a really important part because it is the only way you have to make sure that everything that has been done by the automatic process has actually succeed. There are different kind of tests that may be done at different stage. When working with infrastructure deployment, you probably already do integration testing / smoke testing (i.e. validate that everything seems to be good once it's deployed), load testings to make sure that your infrastructure can handle your users' load and maybe security / penetration testing, to ensure your product is secure enough. It's possible to partially automate some of these tests, or at least some steps of these tests.

In this blog post, I will explain how I have used the [Terratest framework](https://github.com/gruntwork-io/terratest/) to automate the validation of the infrastructure that has been deployed with Terraform.

*Note: there are other options for doing testing, but Terratest was the one that we've choosed to use because it seemed to be the easiest to use and the more robust.*

Terratest is an open source framework that basically allows to execute a Terraform deployment and then write some validation tests using the Go language, before destroying everything. This is really platform integration test, infrastructure is going to be deployed for real against your target platform (Microsoft Azure, in this case) while the tests will be execute. I really like the flexibility that Terratest offers: it deals with all the Terraform stuff for you, and give you the hand to execute any Go code to test that everything in the deployment went fine!

## How it works?

Terratest is really easy to use:

```go
terraformOptions := &terraform.Options {
  // The path to where your Terraform code is located
  TerraformDir: "../tf",
}

// At the end of the test, run `terraform destroy` to clean up any resources that were created
defer terraform.Destroy(t, terraformOptions)

// This will run `terraform init` and `terraform apply` and fail the test if there are any errors
terraform.InitAndApply(t, terraformOptions)

// execute your tests
checkAndValidateInfrastructureDeployment(t, terraformOptions)
```

Basically in the snippet above, I've declared a variable that stores where the Terraform code I want to test is located. Then, I defer the call to `terraform destroy` to make sure it's called after all my code below is executed. The `terraform.InitAndApply` function call is responsible for initializing Terraform in the tested directory, downloading all the plugins / providers dependencies and do the `terraform apply` to deploy the infrastructure.

Finally, the last function `checkAndValidateInfrastructureDeployment` is responsible for executing the test once the infrastructure is deployed and before it is destroyed. You can run any code in this function. It can for example use the [Azure Go SDK](https://github.com/Azure/azure-sdk-for-go) to make some queries to the Azure APIs, checking for services existance... Or it can use the Kubernetes SDK to try to connect an Azure Kubernetes cluster to make sure it is correctly deployed and has the right number of nodes etc... I will detail a bit more later in this post.

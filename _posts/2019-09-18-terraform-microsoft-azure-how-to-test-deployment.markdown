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

When it comes to automation, testing is a really important part because it is the only way you have to make sure that everything that has been done by the automatic process has actually succeed. There are different kinds of tests that may be done, at different stages. When working with infrastructure deployments, you probably already have done integration testing / smoke testing (i.e. validate that everything seems to be good once it's deployed), load testings to make sure that your infrastructure can handle your users' load and maybe security / penetration testing, to ensure your product is secure enough. It's possible to partially automate some of these tests, or at least some steps of these tests.

In this blog post, I will explain how I have used the [Terratest framework](https://github.com/gruntwork-io/terratest/) to automate the validation of the infrastructure that has been deployed with Terraform.

*Note: if you look on the Internet, there are other options for testing Terraform code, but Terratest was the one that  seems to be the easiest to use and the more robust (IMHO).*

## What is Terratest?

Terratest is an open source framework that allows to execute a Terraform deployment and then write some validation tests using the Go language, before destroying everything. This is really platform integration tests, infrastructure is going to be deployed for real on the target platform (Microsoft Azure, in this case - but Terratest is not specific to Azure) while the tests will be executed. I really like the flexibility that Terratest offers: it deals with all the Terraform stuff for you, and give you the hand to execute any Go code to test that everything in the deployment went fine! And it's fully integrated with Go test, that makes it really easy to use!

## How it works?

Here is a simple Terraform test with Terratest:

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

In the snippet above, I've declared a variable that stores where the Terraform code I want to test is located (`../tf` for example). Then, I defer the call to `terraform destroy` to make sure it's called after all my code below is executed. The `terraform.InitAndApply` function call is responsible for initializing Terraform in the tested directory, downloading all the plugins / providers dependencies and do the `terraform apply` to deploy the infrastructure.

Finally, the last function `checkAndValidateInfrastructureDeployment` is responsible for executing the test once the infrastructure is deployed and before it is destroyed. You can run any code in this function. It can for example use the [Azure Go SDK](https://github.com/Azure/azure-sdk-for-go) to make some queries to the Azure APIs, checking that some resources actually exist... Or it can use the Kubernetes SDK to try to connect an Azure Kubernetes cluster to make sure it is correctly deployed and has the right number of nodes etc...

## Getting started with Terratest

To be able to run Terratest, you need to install some tools on your machine. It is based on the Go language test framework (you can run it using `go test`) so you need to have Go language installed on your machine. You can find all the requirements in the [Quickstart section](https://blog.jcorioland.io/archives/2019/09/04/terraform-microsoft-azure-introduction.html) of the Terratest GitHub repository.

It's also possible to run Terratest into a Docker container, I will come back on this solution in the next post of this series as it is the way I have chosen to be able to execute continuous integration on Azure Pipeline :-)

## Azure Authentication

Terratest is actually using Terraform to deploy the infrastructure to Azure, before running code to test it. Because it uses Terraform directly, you have the exact same [authentication options](https://www.terraform.io/docs/providers/azurerm/auth/azure_cli.html) available than when using Terraform: Azure CLI, Azure Managed Identity, Service Principal + Certificate or Service Principal + Password.

When running Terratest on your development machine, I suggest that you use the same authentication method than you use with Terraform. For example, you can let Terraform use your Azure CLI token. When running in an automated pipeline, Managed Identities (on self-hosted agent, for example) or service principal are better solutions.

## Case study: Azure Kubernetes Service module

Now that you have all the basics about Terratest, that you have it on your machine and that you have understood how it connects to Microsoft Azure, let's see how it is possible to use it to test the [Azure Kubernetes Service module of the reference implementation](https://github.com/jcorioland/terraform-azure-ref-aks-module) that comes with this blog post series!

Look at the [test folder](https://github.com/jcorioland/terraform-azure-ref-aks-module/tree/master/test) of the AKS module repository. It contains several things:

* A `dependencies` folder, that contains the Terraform definition of all the infrastructure that should be already available when the AKS module is deployed (i.e. the core networking)
* A `fixture` folder that contains the Terraform code that we want to test, here the AKS module:

```hcl
module "tf-ref-aks-module" {
  source                          = "../../"
  environment                     = "${var.environment}"
  location                        = "${var.location}"
  kubernetes_version              = "${var.kubernetes_version}"
  service_principal_client_id     = "${var.service_principal_client_id}"
  service_principal_client_secret = "${var.service_principal_client_secret}"
  ssh_public_key                  = "${file("~/.ssh/testing_rsa.pub")}"
}
```

* A Go [code file](https://github.com/jcorioland/terraform-azure-ref-aks-module/blob/master/test/aks_kubeconfig_test.go) that contains the actual code of the test.

The `TestAKSKubeConfig` is the main test function of the file. It's the entry point that will be called when launching the Go test command. It contains all the steps described previously:

The Terraform configuration:

```go
terraformFixtureOptions := &terraform.Options{
    TerraformDir: "./fixture",
    VarFiles:     []string{"testing.tfvars"},
}
```

The Terraform initialization, deployment and destroy:

```go
defer terraform.Destroy(t, terraformFixtureOptions)
terraform.InitAndApply(t, terraformFixtureOptions)
```

The test itself:

```go
aksKubeConfig := terraform.Output(t, terraformFixtureOptions, "aks_kube_config")

err := testKubeConfig(aksKubeConfig)
if err != nil {
    t.Fatalf("AKS KubeConfig test has failed: %e", err)
}
```

What is interesting here is that we retrieve the Kubernetes configuration from the outputs of the AKS module deployment to be able to pass it to the `testKubeConfig` function that is responsible for testing it. This function uses the Kubernetes Go SDK to connect to the deployed cluster and to validate that it has two nodes:

```go
func testKubeConfig(aksKubeConfig string) error {
	// get a bytes array with the AKS kube config
	kubeconfigBytes := []byte(aksKubeConfig)

	// write a temporary file with kube config
	err := ioutil.WriteFile("/tmp/kubeconfig", kubeconfigBytes, 0644)
	if err != nil {
		return err
	}

	// create config from file
	config, err := clientcmd.BuildConfigFromFlags("", "/tmp/kubeconfig")
	if err != nil {
		return err
	}

	// create a new Kubernetes client using the config
	client, err := kubernetes.NewForConfig(config)
	if err != nil {
		return err
	}

	// retrieve the list of nodes
	nodesList, err := client.CoreV1().Nodes().List(metav1.ListOptions{})
	if err != nil {
		return err
	}

	// check there are two nodes
	expectedNodesCount := 2
	actualNodesCount := len(nodesList.Items)
	if actualNodesCount != expectedNodesCount {
		return fmt.Errorf("Expected nodes count = %d, Actual = %d", expectedNodesCount, actualNodesCount)
	}

	return nil
}
```

*Note: you can also notice that the init, apply and destroy sequence is done for the dependencies folder before deploying Azure Kubernetes Service - in that way everything will be created and then removed. It is also possible to just assume that tests are ran in sequence and that everything will be deleted by another process. It's up to you :-)*

## Running the test

The test is ready to be run! This is maybe the simple step of the whole process, because Terratest is integrated with the Go test framework. You just need to make sure that your work is in your Go workspace (GOPATH) and then run:

```bash
# ensure dependencies
dep ensure

# set environment variables
export TF_VAR_service_principal_client_id=$SERVICE_PRINCIPAL_CLIENT_ID
export TF_VAR_service_principal_client_secret=$SERVICE_PRINCIPAL_CLIENT_SECRET

# run test
go test -v ./test/ -timeout 30m | tee test_output.log
terratest_log_parser -testlog test_output.log -outputdir test_output
```

* The command `dep ensure` is responsible for downloading the Go dependencies that your project needs (in that case, Terratest and Kubernetes SDK).
* The two environment variables are specific variables for the Azure Kubernetes Service deployment.
* The `go test` command is actually launching the tests defined into the `test` directory.
* Finally, the `terratest_log_parser` allows to generate an XML file with the test result in the JUnit format. This will help to integrate with Azure DevOps, but again, it will be discussed the next part of the series.

All that you have to do is to wait for the test to be completed. Because the test will really deploy all the infrastructure defined in the dependencies and fixture it can take a while before it is completed.

Here is a (truncated) example of log that you'll get:

```bash
=== RUN   TestAKSKubeConfig
=== PAUSE TestAKSKubeConfig
=== CONT  TestAKSKubeConfig
TestAKSKubeConfig 2019-09-03T08:29:39Z retry.go:72: terraform [init -upgrade=false]
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:87: Running command terraform with args [init -upgrade=false]
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:158: 
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:158: Initializing the backend...
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:158: 
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:158: Initializing provider plugins...
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:158: - Checking for available provider plugins...
TestAKSKubeConfig 2019-09-03T08:29:39Z command.go:158: - Downloading plugin for provider "azurerm" (terraform-providers/azurerm) 1.33.1...
TestAKSKubeConfig 2019-09-03T08:29:41Z command.go:158: 

...

TestAKSKubeConfig 2019-09-03T08:35:20Z command.go:158: module.tf-ref-aks-module.azurerm_kubernetes_cluster.aks: Still creating... [5m0s elapsed]
TestAKSKubeConfig 2019-09-03T08:35:30Z command.go:158: module.tf-ref-aks-module.azurerm_kubernetes_cluster.aks: Still creating... [5m10s elapsed]

...

TestAKSKubeConfig 2019-09-03T08:46:58Z command.go:158: 
TestAKSKubeConfig 2019-09-03T08:46:58Z command.go:158: Destroy complete! Resources: 3 destroyed.
--- PASS: TestAKSKubeConfig (1039.53s)
PASS
ok  	aks-module/test	1039.539s
time="2019-09-03T08:46:58Z" level=info msg="reading from file"
time="2019-09-03T08:46:58Z" level=info msg="Directory /go/src/aks-module/test_output already exists"
time="2019-09-03T08:47:02Z" level=info msg="Directory /go/src/aks-module/test_output already exists"
time="2019-09-03T08:47:02Z" level=info msg="Closing all the files in log writer"
time="2019-09-03T08:47:02Z" level=info msg="Directory /go/src/aks-module/test_output already exists"
total 84
drwxr-xr-x 2 vsts docker  4096 Sep  3 08:47 .
drwxr-xr-x 6 vsts docker  4096 Sep  3 08:28 ..
-rw-r--r-- 1 root root     326 Sep  3 08:47 report.xml
-rw-r--r-- 1 root root      70 Sep  3 08:47 summary.log
-rw-r--r-- 1 root root   65187 Sep  3 08:47 TestAKSKubeConfig.log
```

*Note: you can also have a look to [the tests of the common module](https://github.com/jcorioland/terraform-azure-ref-common-module/blob/master/test/common_module_test.go) that uses the Azure Go SDK to validate that resources have been created correctly.*

## Conclusion

In this blog post, I have explained how you can use the Terratest framework and the Go language to run integration / validation tests of your Terraform deployments. The next post of the series will discuss continuous integration (CI) and more specifically how you can use Docker and Azure Pipeline to run these test each time you update your infrastructure code, as you would do for any application code.

Stay tuned!
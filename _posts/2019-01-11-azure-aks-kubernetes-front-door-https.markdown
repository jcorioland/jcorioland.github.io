---
layout: post
title:  "Using Azure Front Door to handle SSL termination with Azure Kubernetes Service"
date:   2019-01-11 16:20:00 +0200
categories: 
- Microsoft Azure
- Kubernetes
author: 'Julien Corioland'
identifier: '6cd50449-793f-4a34-872d-b877ad7b480b'
---

[Azure Front Door](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-overview) allows to manage web traffic routing at the global level. It has a lot of features like URL-based routing, session affinity, URL rewriting, health probes and also SSL termination.

In this post, I will describe how to setup SSL offloading for your applications running in Azure Kubernetes Service with Azure Front Door.

<!--more-->

  *Note: At writing time, Azure Front Door service is in Preview.*

First, you need to have a Kubernetes cluster up and running, and an application or ingress controller exposed through a public IP address. If you don't have an Azure Kubernetes Service cluster yet, you can create one following [this documentation](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough). If you don't have an application exposed yet, you can deploy a simple Nginx proxy with the following commands:

```bash
# create an nginx deployment
kubectl create deployment --image nginx nginx

#expose the nginx server through a public load balancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

Wait for your public IP address to be available:

```bash
kubectl get svc -w
```

In the Azure portal, create a new resource of type `Azure Front Door`. Once create, you should land on a designer page that helps you to define:

* Frontend hosts: it's the entry point on Azure Front Door from which you want to reach out your application
* Backend pools: it's the entry point(s) of the application(s) that you want to reach out with Azure Front Door
* Routing rules: rules that defines how you want the web traffic to be routed between Azure Front Door and your backend pools

![Azure Front Door designer](/images/aks-afd-ssl/aks-afd-1.png)

Start by adding a new frontend host. You can choose or not to link it to a custom domain. For that sample, I choose one on the `*.azurefd.net` domain. 

Then, you need to add a backend pool: give it a name, configure health probe and load balancing as your require. Then you need to add one or more backend that hosts your application. In that case, choose *Custom Backend* in the list for backend type and fill with the IP address of the application running in Kubernetes:

![Azure Front Door backend pool](/images/aks-afd-ssl/aks-afd-2.png)

Finally, configure the routing rule that needs to apply when web traffic reach out the Azure Front Door service. In that case, you can keep the defautl values in the `Basic` tab:

![Azure Front Door routing rule - Basic](/images/aks-afd-ssl/aks-afd-3.png)

In the advanced tab, you can choose how do you want the traffic to be forwarded. You can match the incoming request, or force HTTP or HTTPS. If you followed my example, force http, as the nginx deployment only listening on http 80.

  *Note: in the case you force http, that means that the connection between Azure Front Door and your application will not be encrypted.*

Validate and create the Azure Front Door service.

Once the Azure Front Door has been created, you should be able to navigate to `https://yourhost.azurefd.net` and reach out your application. It can take a few minutes for the service to be fully available.

Hope this helps!

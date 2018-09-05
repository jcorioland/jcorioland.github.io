---
layout: post
title:  "What's new with Docker for Windows and Microsoft Azure users ?"
date:   2016-03-29 10:00:00 +0200
categories: 
- Microsoft Azure
- Docker
author: 'Julien Corioland'
identifier: 'dcb3b8dc-1f59-413d-81c5-a6582fa2d5ba'
---

Maybe you have missed the last annoucements that have been made about Docker last week, but there are few things that may interest you if you’re running Windows or working with Microsoft Azure !

<!--more-->

### Docker Machine Azure Driver

A new Docker Machine driver for Microsoft Azure has been release. You can read the announcement [here](https://azure.microsoft.com/en-us/blog/docker-machine-azure-driver/).

Docker Machine is a really simple-to-use tool that allows to create Docker hosts on different virtualization systems (Hyper-V or Virtual Box, for example) or Cloud Providers (Azure or Digital Ocean, for example).

This new Microsoft Azure driver will be shipped with the next version of Docker Machine (0.7.0) but you can get it now from the [Docker GitHub Releases page](https://github.com/docker/machine/releases/) (as RC1). It is a major update of the driver that now uses the Azure Resource Manager deployment model, instead of ASM. Once installed, just ask Docker Machine to create a new host with this driver and it will handle all the technical stuff for you.
Check you are running the 0.7.0-RC1 version using `docker-machine --version`:

![Docker Machine](/images/msdn-archives/docker-azure-01.png)

Then you have to get the id of the Azure subscription where you want to create the host. You can get it from the Azure portal, in the Subscriptions section:

![Azure Portal](/images/msdn-archives/docker-azure-02.png)

The command to create the Docker host is:

```bash
docker-machine create –d azure --azure-subscription-id <your_sub_id> <vm_name>
```

![Docker Machine](/images/msdn-archives/docker-azure-03.png)

> Note: When running the command for the first time on an Azure subscription you will be asked to use Azure Device Login with a code to let Docker Machine administrate your subscriptions. Just go on [https://aka.ms/devicelogin](https://aka.ms/devicelogin) and enter the code provided by docker machine.

All the other options have default values, but you can customize almost everything if you want to control the name of the resource group, availability sets… To see all the options, type `docker-machine create –d azure --help`:

![Docker Machine](/images/msdn-archives/docker-azure-04.png)

After the command has ended you can go on the Azure portal and display the resources that have been created by docker machine:

![Azure Portal](/images/msdn-archives/docker-azure-05.png)

Now, run the command `docker-machine env <vm_name>` to display the environment variable that you have to set to connect your Docker host:

![Docker Machine](/images/msdn-archives/docker-azure-06.png)

> Note: the last line gives you the command you have to type to set all the environment variables in your system.

That’s it ! You can type docker info and you should see that you are connected to the remote Azure host.

### Docker for Windows

Docker has just released its new Windows and Mac applications (in replacement for Docker Toolbox)! You can see the announcement [here](https://blog.docker.com/2016/03/docker-for-mac-windows-beta/).

As you probably now, it is not possible to run a Docker host on Windows client or Mac OS. This new application allows to get a development environment ready in a few minutes. As it is in private beta you should ask Docker for an access on [this page](https://beta.docker.com/).

> Note: the Windows application is currently available only for Windows 10 and you should activate Hyper-V before the installation.

Once installed, wait a minute while your environment is configuring. During this time, Docker for Windows app will create a virtual machine that runs a small Linux distribution “Moby alpha” (based on Alpine) in your local Hyper-V. It will also configure all the environment variables on your system so you can easily use docker, docker-compose or docker-machine from the command prompt and Powershell.

As soon as the configuration is completed, you can open a Powershell window and type `docker info`:

![Docker for Windows](/images/msdn-archives/docker-azure-07.png)

And voilà !

If you are looking for the application, check your notification area of your taskbar:

![Docker for Windows](/images/msdn-archives/docker-azure-08.png)

You can also read [this post from Steve Lasker](https://blogs.msdn.microsoft.com/stevelasker/2016/03/24/docker-for-windows-beta-released/) that details its top 6 features for this new application.

Enjoy Docker on Windows and Microsoft Azure !
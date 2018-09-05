---
layout: post
title:  "Create and run Hyper-V containers using Docker on Windows 10 desktop"
date:   2016-05-31 10:00:00 +0200
categories: 
- Docker
author: 'Julien Corioland'
identifier: '8d737cf3-0c27-442d-b0b2-ae443cc8af55'
---

As you probably know the Windows Insider program allows to get preview Windows 10 builds to test the new features that are coming in the next major update of Windows. Since a few weeks, a new feature named **Container** has been included in Windows 10 preview builds. This feature brings Hyper-V containers on the desktop, natively.

> Note: The following will work only on Windows 10 Professional and Enterprise.

<!--more-->

There are two different kinds of containers on Windows : Windows Container and Hyper-V Container. They work in the same way, instead of that Hyper-V containers are more isolated than Windows Container because they are running in a very lightweight virtual machine that provides kernel isolation and not just process isolation.

For more information about Hyper-V containers, check the official documentation on [MSDN](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/management/hyperv_container)

You want to try it? It’s pretty simple! First, you need to enroll your machine in the Windows Insider program (see link above) and get the latest build from the insider fast ring.

Once done, just open the **Turn Windows features on or off**, then select **Containers** and **Hyper-V** in the list:

![Turn On Off Features](/images/msdn-archives/hyperv-containers-windows10-01.png)

Click OK to install the two components and restart your computer when the installation has completed.

Now, open a PowerShell console in administrator mode. The first thing to do is to change the execution policy to unrestricted using the following command:

```powershell
Set-ExecutionPolicy Unrestricted
```

Then you can install the ContainerImage package provider:

```powershell
Install-PackageProvider ContainerImage -Force
```

![PowerShell](/images/msdn-archives/hyperv-containers-windows10-02.png)

This package provider will allow you to pull the base operating system image to run Hyper-V containers. In this case, as you are going to run Hyper-V container you need to use the NanoServer base container image. To pull this image, you have to execute the following command:

```powershell
Install-ContainerImage NanoServer
```

Depending on your Internet connection, this step can take a while…

Once the base container image is downloaded you can install [Docker](http://www.docker.com/) on your machine to be able to run and manage containers. To do that, you can download a PS script from [http://aka.ms/tp5/update-containerhost](http://aka.ms/tp5/update-containerhost) and save it on your computer. Just run this script that will install all the stuff you need: the Docker client, the Docker deamon, all the configuration, environment variables…

![PowerShell](/images/msdn-archives/hyperv-containers-windows10-03.png)

You can now use the Docker commands to work with Hyper-V containers! For example getting the list of available images on your computer:

![PowerShell](/images/msdn-archives/hyperv-containers-windows10-04.png)

To be able to use this image without specifying its tag you can tag it as the latest image using the `docker tag` command:

![PowerShell](/images/msdn-archives/hyperv-containers-windows10-05.png)

Now you can switch to a CMD window, with administrator rights and create a new Hyper-V container use the following command:

```powershell
docker run –it --isolation=hyperv nanoserver cmd
```

After a few seconds you will be running in an Hyper-V container!

![PowerShell](/images/msdn-archives/hyperv-containers-windows10-06.png)

You’re done!  You can now run any Hyper-V container on your Windows 10 computer. 

Do you like it?
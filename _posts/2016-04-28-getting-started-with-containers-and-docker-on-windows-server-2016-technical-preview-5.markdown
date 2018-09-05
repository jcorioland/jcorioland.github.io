---
layout: post
title:  "Getting started with containers and Docker on Windows Server 2016 Technical Preview 5"
date:   2016-04-28 10:00:00 +0200
categories: 
- Microsoft Azure
- Docker
author: 'Julien Corioland'
identifier: 'f9438bb9-4ff2-49a6-8723-20aafef12b1a'
---

The Technical Preview 5 of Windows Server 2016 has been released yesterday, and it comes with a lot of new cool things for Windows containers and Docker !

To get started with this new preview, you have two solutions:

- Download it from [this page](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-technical-preview) and run it where you want.
- Deploy a new VM in Azure from the MarketPlace

<!--more-->

I have chosen the second option, as it is simple and fast. Just search for Windows Server 2016 in the Azure MarketPlace and you’re done:

![Marketplace](/images/msdn-archives/get-started-docker-windows-01.png)

Once you are connected on your machine, you need to configure it to be a container host. It can be done with a few steps:

```powershell
Install-WindowsFeature Containers
```

Once installed, you need to restart the machine using:

```powershell
New-Item -Type Directory -Path 'C:\Program Files\docker\'
Invoke-WebRequest https://aka.ms/tp5/b/dockerd -OutFile $env:ProgramFiles\docker\dockerd.exe 
Invoke-WebRequest https://aka.ms/tp5/b/docker -OutFile $env:ProgramFiles\docker\docker.exe
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\Docker", [EnvironmentVariableTarget]::Machine)
```

The commands above create a new Docker folder in Program Files and download the docker deamon and client binaires in this directory. The last one adds the directory in the Path environment variable.
The Docker deamon now supports an option to register it as a service on the machine:

```powershell
dockerd --register-service
```

To start the docker service use:

```powershell
Start-Service docker
```

Now, you need to download the base image for Docker containers on Windows. You can do it with the following commands:

```powershell
Install-PackageProvider ContainerImage -Force
Install-ContainerImage -Name WindowsServerCore
```

Restart the Docker service:

```powershell
Restart-Service docker
```

And your done!

```powershell
docker info
```

![Docker Windows](/images/msdn-archives/get-started-docker-windows-02.png)

You can also get the list of available images on your system by typing:

```powershell
docker images
```

![Docker Windows](/images/msdn-archives/get-started-docker-windows-03.png)

As you can see, an image named windowsservercore is available, but it does not have the “latest” tag. To tag it, type:

```powershell
docker tag windowsservercore:10.0.14300.1000 windowsservercore:latest
```

![Docker Windows](/images/msdn-archives/get-started-docker-windows-04.png)

You can run your first container with the following command:

```powershell
docker run -it windowsservercore cmd
```

It will run a new container in interactive mode and execute cmd inside the container:

![Docker Windows](/images/msdn-archives/get-started-docker-windows-05.png)

Type `exit` to leave the container.

As [announced by Docker](https://blog.docker.com/2016/04/docker-windows-server-tp5/), it is now possible to write Dockerfile for Windows-based images and you can start to push/pull Windows images to and from the Docker Hub. Some example of Windows-based Dockerfile are available on GitHub: [https://github.com/Microsoft/Virtualization-Documentation/tree/master/windows-container-samples/windowsservercore](https://github.com/Microsoft/Virtualization-Documentation/tree/master/windows-container-samples/windowsservercore)

For example, you can use this Dockerfile to create an image that will allow to run IIS in a container:

```Dockerfile
# This dockerfile utilizes components licensed by their respective owners/authors.
FROM windowsservercore
MAINTAINER neil.peterson@microsoft.com
LABEL Description="IIS" Vendor="Microsoft" Version="10"
RUN powershell -Command Add-WindowsFeature Web-Server
CMD [ "ping localhost -t" ]
```

Save this Dockerfile in an empty directory. Open a PowerShell window in this directory and build the container image with the `docker build` command:

```powershell
docker build -t yourimagename .
```

![Docker Windows](/images/msdn-archives/get-started-docker-windows-06.png)

Once the image is built, login to your Docker Hub account using the `docker login` command:

![Docker Windows](/images/msdn-archives/get-started-docker-windows-07.png)

And push the image using the `docker push` command:

![Docker Windows](/images/msdn-archives/get-started-docker-windows-08.png)

Same experience than Docker with Linux containers ! It’s awesome !

Here are some resources that you may be interested in to go deeper in Windows Containers:

- Windows Containers overview: [https://msdn.microsoft.com/en-us/virtualization/windowscontainers/about/about_overview](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/about/about_overview)
- The Containers Channel on Channel9: [https://channel9.msdn.com/Blogs/containers](https://channel9.msdn.com/Blogs/containers)
- Windows Server Containers – Quick Start using Docker: [https://msdn.microsoft.com/en-us/virtualization/windowscontainers/quick_start/manage_docker](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/quick_start/manage_docker)
- Dockerfile reference on Windows: [https://msdn.microsoft.com/en-us/virtualization/windowscontainers/docker/manage_windows_dockerfile](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/docker/manage_windows_dockerfile)
- Windows Dockerfile samples on GitHub: [https://github.com/Microsoft/Virtualization-Documentation/tree/master/windows-container-samples/windowsservercore](https://github.com/Microsoft/Virtualization-Documentation/tree/master/windows-container-samples/windowsservercore)
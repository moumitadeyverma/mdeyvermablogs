# Migration journey from Azure Service Fabric(ASF) Containers to Azure Kubernetes Service(AKS)  
This article is a direct recount of migration journey from ASF - containers to AKS containers for a 8 years old legacy app.
<br>
When we had just started out, we could not find appropriate guidelines or steps to ease me through the process. I hope this article will help those of you planning a similar journey to AKS.
<br>
Journey began when one of my costumers got the [communication](https://techcommunity.microsoft.com/t5/containers/reminder-updates-to-windows-container-runtime-support/ba-p/3620989) that  for their stable workloads deployed in ASF containers they have to face service disruption. ASF windows containers were internally using Mirantis Container Runtime, former DockerEE.
<br>
**"Post 30 April 2023 Service Fabric customers using “with containers” VM images will face service disruptions as Microsoft will remove the “with container” VM images from the Azure image gallery".**
<br>After initial investigation we figured out that moving to AKS Linux container is in the final roadmap of the current product.
<br>So we started our journey with two types of containerized workloads **Windows and Linux** before they reach the final goal.

## Decision Factors
The code is legacy and we didn't have enough time to rewrite the code within a short duration from .net framework to .net core.
<br>
Also we had to choose the highest common supported .NET version for win 2019 and win 2022. Since ASF workloads were still modified and deployed parallely in ASF using win 2019.So we chose the latest common version 4.8. [know more](https://github.com/microsoft/dotnet-framework-docker/issues/849)
<br>
#### We took the below approach for framework upgrade
  a.	Upgraded from .net framework current version to the latest version 4.8 and deployed in AKS **Windows** containers. 
  <br>
  b.	Upgraded from .net core to .net 6.0 and deployed in AKS **Linux** containers.

Support for .net framework 4.6.1 ended on april 2022 [know more](https://devblogs.microsoft.com/dotnet/net-framework-4-5-2-4-6-4-6-1-will-reach-end-of-support-on-april-26-2022/).
For deciding  the .net framework version use the [link](https://learn.microsoft.com/en-us/lifecycle/products/microsoft-net-framework)
<br>
For upgrading to latest version use the below tools-

 **[Portability Analyzer](https://docs.microsoft.com/en-us/dotnet/standard/analyzers/portability-analyzer)**
 <br>
 **[Upgrade Assistant](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant/tutorial/install-upgrade-assistant)** 
 <br>
 
#### We took the below approach for containers upgrade
• Moved existing windows containers to latest AKS windows containers- WS2022 for a huge number of important reasons [know more](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/whats-new-ws2022-containers)
<br>
• Moved the existing linux containers to AKS Linux containers 
<br>
• Left as is the WCF services hosted in ASF with worker role and customization. [know more](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-communication-wcf).
Since customer is planning eventually to move everything to linux containers ,so later they have to re implement the WCF part with more modern technologies like gRPC or http Request/Response webApIs.
<br>
Refer to the below links to find out the correct base image, that has IIS Roles enabled for windows containers
<br>
https://mcr.microsoft.com/en-us/product/dotnet/framework/aspnet/about
<br>
https://hub.docker.com/_/microsoft-windows-servercore-iis?tab=description
<br>
## Developer steps
Changes are to be made in the new or existing repository as per the organization process.

#### Discover your Workloads: 
Use tools like [Cloud Pilot](https://www.youtube.com/watch?v=cO27cq9IiuE&list=PL8ZFxfkhI2ewuyjmmcls7_xTJX1ciEARl) to discover applications and APIs that are in scope of migrtaion.
<br>
Make the workloads compatible, by checking the reports provided by the tool-
<br>
  • Recommendations Rresults  by 4 pillars- Application & Platform design, Security,Network & Availability and Storage
<br>
  • Migration Effort.
<br>
  • Rediness Status in percentage.
<br>
  • Detailed code level fix.

#### Migration from Framework 4.6.1 to 4.8
•	Change the target framework from “<TargetFrameworkVersion>v4.6.1</ TargetFrameworkVersion >“ to “<TargetFrameworkVersion >v4.8</ TargetFrameworkVersion >“ for every project in the solution that you are targeting.
<br>
•	If your repo has multiple solutions and they are configured to build the pipeline, then update them all together to use the new version. 
<br>
•	Some NuGet packages will give issues, reinstall them again, fix the issues locally. It will not affect the pipeline build because the NuGet is installed every time the pipeline runs.
<br>
•	If the AKS deployment folder is not available, create and add the yaml files to the AKS deployment folder. Change Docker image for latest version. 
<br>
•	In Docker file you may get few errors related to "Import-Module WebAdministration" in POWERSHELL command. Check the [reference link](https://docs.docker.com/engine/reference/builder/#shell). If the Web server Role (IIS) is not active/installed, you need to enable the Web Role before "Import-Module WebAdministration" like this.
<br>
  run powershell Install-WindowsFeature Web-Server,Web-Common-Http,Web-Mgmt-Console -Restart
<br>
•	Consider Stateless microservices.Stateful microservices are not in scope of migration.
<br>
•	Data processing logic using Persistent Storage must be validated. Use Volumes, Persistent Volumes, Storage Classes, Persistent volume claims.
<br>
•	AKS container hosted sites interact with ASF hosted WCF services.
<br>
•	Changes related to Common dLL will be required to be published first (nuget package /physical dll references)
<br>
•	The Service Fabric programming models like reliable services, reliable actors, Guest executables will need revisit or redesign.

#### DevOps practices
•	All developers to install the latest version of Visual Studio 2022 and appropriate .NET SDK. 
<br>
•	While building the solution locally, you may face issues related to NuGet not able to install after the upgrade, just restart the visual studio and load the solution again.
<br>
•	If project dlls are pushed to common folder, the consumer project will fail until the related common DLL folders are having correct versions. Use own [NuGet](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-nuget-registry) feeds/GitHub Package Registry
<br>
•	You may find “DLL not found issue” with the upgrade, to solve this please remove the related binding from the web.config and run the build again.

#### Code Changes 
•	Add K8s.deployment folder in Repo.
<br>
•	Change Environment variable to Config.yaml (refer to Service manifest used for ASF all used env variables from service manifest file must be moved to config.yaml)
<br>
•	If you are using 2 docker files to run simultaneously ASF & AKS, then do the below changes to upgrade image to support 4.8 
<br>
     o Add new Docker file for AKS.<br>
     o Change existing Docker file for ASF till it is not migrated to AKS production.
<br>
•	Use latest Windows server 2022 Images.[refernce link](https://learn.microsoft.com/en-us/azure/aks/upgrade-windows-2019-2022)
<br>
#### Pipeline Changes 
•	Add AKSConfigPrefix in Web.config
<br>
•	Do ADO pipeline changes related to .NET FrameWork for Windows.Also check default agent pools supported versions.
<br>
•	Do Octopus Pipeline Changes Windows , Octopus Deploy Linux - Octopus Deploy.
<br>
•	All pipelines should be independently released so the apps team can take release any time. 
<br>
•	Disable DEV ADO and Octopus pipeline once they are good with aks
<br>
•	Carefully manage the dll chains. Use NuGet for package management.
<br>
•	Application team to communicate with all consumers about the New dll versions. Publish the documented process to consume the latest dlls for new changes. 
<br>
#### AKS deployment related Changes
o	Decide the AKS version as per [release calander](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli#aks-kubernetes-release-calendar).
<br>
o In AKS if you need Windows Containers node pool, then while creating the cluster you need bydefault a system linux container node pool with at least three nodes.In the AKS cluster you have to use the "y --windows-admin-username parameter" otherwise AKS will not prepare itself for windows nodepools.[refernce  link](https://learn.microsoft.com/en-us/azure/aks/learn/quick-windows-container-deploy-cli#create-an-aks-cluster)
<br>
o	Next create windows nodepool [refernce link](https://learn.microsoft.com/en-us/azure/aks/learn/quick-windows-container-deploy-cli#add-a-windows-server-2022-node-pool)
** you may need to update the CLI version to 2.40, with terraform deployment.
<br>
o	Check Resource Quotas/RBAC at namespace level
<br>
o Configure [scaling](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)
<br>
o	Review existing policy to handle AKS Configuration, maintain specific set of node type, number etc
<br>
o	Maintain segregation of Node pool- system & user, use combination of taints/tolerations, node affinity + pod affinity in specified namespace for user node pool
<br>
o	Use SLA backed AKS
<br>
o	Use secrets from keyvault with CSI driver if they are used through config.yaml
<br>
o	Optimize request and limit at pod level, after few deployments and testing are completed.
<br>
o	Communication from namespace A to namespace B can be restricted through network policies
<br>
o	Use ACR as image repository and [authenticate](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication?tabs=azure-cli)
<br>
## Infra Changes 
•	Create New Subnet for AKS inside the existing Vnet. Subnet size-should be greater or equal to number of pods
<br>
•	Create New AKS Cluster 
<br>
•	Create node pool with ephemeral disk and availability zones. Test throughput
<br>
•	Avoid using custom dns
<br>
•	Use Static public IP address
<br>
•	Use Azure vNET integration via Azure CNI
<br>
## Sample Migration Process for Projects
<br>
**For Windows  container deployment-**
<br>
The DevOps team can perform repository-wise migration. For win framework projects update the framework version with 4.8 on the existing dev repository. 
<br> Once the package is ready, deploy it. Post deployment, point the application on dev branch from ASF to AKS. Once the Developer provides sign-off on Dev, perform API health check. DevOps then updates the Release branch code with the latest framework and deploys the AKS and ASF on QC environment. In between, if any features or bug fixing is required by the app team, then they can work on the current repository and take release at any time. The old Deployment pipeline can be alive till production. Disable the Old Pipeline across all branch’s postproduction deployment.
<br>
**For Linux  container deployment-**
Steps are more or less similar for.NET Core 2.1/2.2/3.1 to .NET 6.0. We used the ubuntu image Ubuntu 22.04 for AKS 1.23. 
<br>
From AKS 1.25 and afterwards, the underlying image will change [refrence link](https://github.com/Azure/AKS/releases/tag/2022-09-25)
<br>
Few more links
* [ASP.NET app containerization and migration to Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/migrate/tutorial-app-containerization-aspnet-kubernetes)
* [Continuation of container support in Azure Service Fabric](https://github.com/Azure/Service-Fabric-Troubleshooting-Guides/blob/master/Deployment/Mirantis-Guidance.md)
## Documentation challenges faced
•	Multiple build related errors during each project migration. Create KB article an document learning for others.
<br>
•	Legacy dll consumption causing more inter dependency and migration is not straight forward without checking them. Use nuget package library/GH Nuget Registry.

## Performance Optimization
With latest deployment in AKS, We performed various performance tests and used multiple tools.The outcome of these performance test activities showed whether there is a overall performance problem or it is in one of the deployments. 
<br>
Best practices
<br>
•	Study network diagram of all components for comparing the two measurable deployments (old and new)
<br>
•	Study conceptional diagram of all workloads, tech stack per workload and where they run on (APIs, web fronts etc) for the two measurable deployments (old and new)
<br>
•	Check test case scenario and the sample data used for performance test (how many concurrent users, how many test scenarios, iterations of test scenarios, sample data either random, or pre-recorded etc). 
<br>
•	If you observe that there are big variations in some operations (small, acceptable averages but big/slow perentiles90). Understand if these variations happen, randomly, or because of sample data but in a deterministic way, or because of caching etc. 
<br>
•	Dig into the full call stack of the calls noted as "External Call" in tools. 
<br>
•	Run the performance test for the environment (with the same sample data, and the same load) at least three times, to check whether there are significant variations per performance test cycle for the same environment (old, or new)
<br>
•	Gather Network stack traces and Process Dumps for given containers during the stress test (And not while manual / user testing). Find a way to maximize duration of dumps.Capture [GCdump](https://devblogs.microsoft.com/dotnet/collecting-and-analyzing-memory-dumps/).capture [tcpdump](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/packet-capture-pod-level) from the pod level.
<br>
•	Use the monitoring tabs in the Azure portal, also connect to the nodes/pods and collect logs [know more](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/identify-memory-saturation-aks)
• If in scope then guide customer to check code base as well.

Feedback appreciated ....__[By Moumita Dey Verma](https://www.linkedin.com/in/moumita-dey-verma-8b61692a/)__



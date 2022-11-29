# Migration journey from Azure Service Fabric(ASF) Containers to Azure Kubernetes Service(AKS)  
This article is a direct recount of migration journey from ASF - containers to AKS containers for a 8 years old legacy app.
<br>
When we had just started out, we could not find appropriate guidelines or steps to ease us through the process. I hope this article will help those of you planning a similar journey to AKS.
<br>
Journey began when one of our costumers got the [communication](https://techcommunity.microsoft.com/t5/containers/reminder-updates-to-windows-container-runtime-support/ba-p/3620989) that  for their stable workloads deployed in ASF containers they have to face issues. ASF windows containers were internally using Mirantis Container Runtime, former DockerEE.
<br>
> **"Post 30 April 2023 Service Fabric customers using “with containers” VM images will face service disruptions as Microsoft will remove the “with container” VM images from the Azure image gallery".**

<br>After initial investigation we figured out that moving to AKS Linux container is in the final roadmap of the current product.
<br>
## Decision Factors
The code is legacy and we didn't have enough time to rewrite the code within a short duration from .net framework to .net core and deploy all in Linux container.So we started our journey with combination of two types of workloads **Windows and Linux**.
<br>
Also we had to choose the highest common supported .NET version for win 2019 and win 2022. Since ASF workloads were still modified and deployed in parallel in ASF using win 2019, we chose the latest common version 4.8. [know more](https://github.com/microsoft/dotnet-framework-docker/issues/849)
<br>
#### We took the below approach to upgrade framework
  - Upgraded from .net framework current version to the latest version 4.8 and deployed in AKS **Windows** containers.(Approx 7x projects)
  * Upgraded from .net core to .net 6.0 and deployed in AKS **Linux** containers. (Approx x projects)

Support for .net framework 4.6.1 ended in April 2022 [know more](https://devblogs.microsoft.com/dotnet/net-framework-4-5-2-4-6-4-6-1-will-reach-end-of-support-on-april-26-2022/).
For deciding  the .net framework version use the [link](https://learn.microsoft.com/en-us/lifecycle/products/microsoft-net-framework)
<br>
For upgrading to latest version use the below tools-

 **[Portability Analyzer](https://docs.microsoft.com/en-us/dotnet/standard/analyzers/portability-analyzer)**,
 **[Upgrade Assistant](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant/tutorial/install-upgrade-assistant)** 
 <br>
 
#### We took the below approach to upgrade containers
  - Moved existing windows containers to latest AKS windows containers- WS2022 for several number of important reasons [know more](https://learn.microsoft.com/en-us/virtualization/windowscontainers/about/whats-new-ws2022-containers)
  * Moved the existing linux containers to AKS Linux containers 
  - Left the WCF services untouched, hosted in ASF with worker role and customization. [know more](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-communication-wcf).
  Since customer is planning eventually to move everything to linux containers ,so later they have to re implement the WCF part with more modern technologies like gRPC or http Request/Response webApIs.
<br><br>Refer to the below links to find out the correct base image, that has IIS Roles enabled for windows containers
https://mcr.microsoft.com/en-us/product/dotnet/framework/aspnet/about
<br>https://hub.docker.com/_/microsoft-windows-servercore-iis?tab=description

## Developer steps
Changes are to be made in the new or existing repository as per the organization process.

#### Discover your Workloads: 
Use tools like [Cloud Pilot](https://www.youtube.com/watch?v=cO27cq9IiuE&list=PL8ZFxfkhI2ewuyjmmcls7_xTJX1ciEARl) to discover applications and APIs that are in scope of migrtaion.
<br>
Make the workloads compatible, by checking the reports provided by the tool-

  - Recommendations Rresults  by 4 pillars- Application & Platform design, Security,Network & Availability and Storage
  * Migration Effort.
  - Readiness Status in percentage.
  * Detailed code level fix guidance.

#### Migration from Framework 4.6.1 to 4.8
  - Change the target framework from “<TargetFrameworkVersion>v4.6.1</ TargetFrameworkVersion >“ to “<TargetFrameworkVersion >v4.8</ TargetFrameworkVersion >“ for every project in the solution that you are targeting.
  - Open the Package Manager Console and run the following command:
> PM> Update-Package -Reinstall
  - If your repo has multiple solutions and they are configured to build the pipeline, then update them all together to use the new version. 
  - Some NuGet will give errors, reinstall them again, fix the issues locally. It will not affect the pipeline build because the NuGet is installed every time the pipeline runs.
  - In Package.Config change the <httpRuntime> tag and add/change the "targetframework ="4.8""
  - The target framework moniker (TFM) to apply when installing the package. This is initially set to the project's target when a package is installed. As a result, different <package> elements can have different TFMs. [Know more](https://learn.microsoft.com/en-us/nuget/reference/target-frameworks)
  - If the AKS deployment folder is not available, create and add the yaml files to the AKS deployment folder. Change Docker image for latest version. 
  - In Docker file you may get few errors related to "Import-Module WebAdministration" in POWERSHELL command. Check the [reference link](https://docs.docker.com/engine/reference/builder/#shell). If the Web server Role (IIS) is not active/installed, you need to enable the Web Role before "Import-Module WebAdministration" like this.
  > run powershell Install-WindowsFeature Web-Server,Web-Common-Http,Web-Mgmt-Console -Restart
  - Consider Stateless microservices.Stateful microservices are not in scope of migration.
  - Data processing logic using Persistent Storage must be validated. Use Volumes, Persistent Volumes, Storage Classes, Persistent volume claims.
  - AKS container hosted sites interact with ASF hosted WCF services.
  - Changes related to Common dLL will be required to be published first, to be consumed by others.
  - The Service Fabric programming models like reliable services, reliable actors, Guest executables will need revisit or redesign.

#### DevOps practices
  - All developers to install the latest version of Visual Studio 2019 (or greater) and appropriate .NET SDK. 
  - While building the solution locally, you may face issues related to NuGet not able to install after the upgrade, just restart the visual studio and load the solution again.
  - If project dlls are pushed to common folder, the consumer project will fail until the related common DLL folders are having correct versions. Use [NuGet](https://learn.microsoft.com/en-us/azure/devops/artifacts/get-started-nuget?view=azure-devops&tabs=windows).
  - You may find “DLL not found issue” with the upgrade, to solve this please remove the related binding from the web.config and run the build again.

#### Code Changes 
  - Add K8s.deployment folder in Repo.
  - Change Environment variable to Config.yaml (refer to Service manifest used for ASF all used env variables from service manifest file must be moved to config.yaml)
  - If you are using 2 docker files to run simultaneously ASF & AKS, then do the below changes to upgrade image to support 4.8 
       * Add new Docker file for AKS.
       * Change existing Docker file for ASF till it is not migrated to AKS production.
  -	Use latest Windows server 2022 [images](https://learn.microsoft.com/en-us/azure/aks/upgrade-windows-2019-2022).
  - Add required yaml files for AKS.

#### Pipeline Changes 
  - Add AKSConfigPrefix in Web.config
  - Do ADO pipeline changes related to .NET FrameWork for Windows.Also check default agent pools supported versions.
  - Do Octopus Pipeline Changes Windows , Octopus Deploy Linux - Octopus Deploy.
  - All pipelines should be independently released so the apps team can take release any time. 
  - Disable DEV ADO and Octopus pipeline once they are good with aks
  - Store andmanage the DLL versions.
  - Application team to communicate with all consumers about the New dll versions. Publish the documented process to consume the latest dlls for new changes. 

#### AKS deployment related Changes
  - Decide the AKS version as per [release calander](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions?tabs=azure-cli#aks-kubernetes-release-calendar).
  - In AKS if you need Windows Containers node pool, then while creating the cluster you need bydefault a system linux container node pool with at least three nodes.In the AKS cluster you have to use the "y --windows-admin-username parameter" otherwise AKS will not prepare itself for windows nodepools.[know more](https://learn.microsoft.com/en-us/azure/aks/learn/quick-windows-container-deploy-cli#create-an-aks-cluster)
  - Next create [windows nodepool](https://learn.microsoft.com/en-us/azure/aks/learn/quick-windows-container-deploy-cli#add-a-windows-server-2022-node-pool)
      * You may need to update the CLI version to higher version (2.40) with terraform deployment.
  - Check Resource Quotas/RBAC at namespace level
  - Configure [scaling](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)
  - Review existing policy to handle AKS Configuration, maintain specific set of node type, number etc
  - Maintain segregation of Node pool- system & user, use combination of taints/tolerations, node affinity + pod affinity in specified namespace for user node pool
  - Use SLA backed AKS
  - Use secrets from keyvault with CSI driver if they are used through config.yaml
  - Optimize request and limit at pod level, after few deployments and testing are completed.
  - Communication from namespace A to namespace B can be restricted through network policies
  - Use ACR as image repository and [authenticate](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication?tabs=azure-cli)

## Infra Changes 
  - Create New Subnet for AKS inside the existing Vnet. Subnet size-should be greater or equal to number of pods.
  - Create New AKS Cluster.
  - Consider using Ephemeral OS disk (faster IOPS). Test throughput.
  >  Specify the ephemeral disks on the AKS cluster at creation time or when adding a new nodepool by specifying the following property:
  >  <br>[--node-osdisk-type {Ephemeral, Managed}]
  - Avoid using custom dns.
  - Use Static public IP address.
  - Use Azure vNET integration via Azure CNI
  - Considering using [Proximity Placement Group](https://learn.microsoft.com/en-us/azure/aks/reduce-latency-ppg#add-a-proximity-placement-group-to-an-existing-cluster).

## Sample Migration Process for Projects

#### For Windows  container deployment
<br>
The DevOps team can perform repository-wise migration. For win framework projects update the framework version with 4.8 on the existing dev repository. 
<br> Once the package is ready, deploy it. Post deployment, point the application on dev branch from ASF to AKS. Once the Developer provides sign-off on Dev, perform API health check. DevOps then updates the Release branch code with the latest framework and deploys the AKS and ASF on QC environment. In between, if any features or bug fixing is required by the app team, then they can work on the current repository and take release at any time. The old Deployment pipeline can be alive till production. Disable the Old Pipeline across all branch’s postproduction deployment.

#### For Linux  container deployment
Steps are more or less similar for.NET Core 2.1/2.2/3.1 to .NET 6.0. We used the ubuntu image Ubuntu 22.04 for AKS 1.23. 
<br>
From AKS 1.25 and afterwards, the underlying image will change.[know more](https://github.com/Azure/AKS/releases/tag/2022-09-25)
<br>
<br>
Few more links-
<br>
[ASP.NET app containerization and migration to Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/migrate/tutorial-app-containerization-aspnet-kubernetes)
<br>
[Continuation of container support in Azure Service Fabric](https://github.com/Azure/Service-Fabric-Troubleshooting-Guides/blob/master/Deployment/Mirantis-Guidance.md)
<br>
## Documentation challenges faced
  - Multiple build related errors during each project migration. Create KB article to help other team members to reduce time for troubleshooting.
  - Legacy dll consumption causing more inter dependency and migration is not straight forward without checking them. Use NuGet.
 
## Performance Optimization
With latest deployment in AKS, We performed various performance tests and used multiple tools.The outcome of these performance tests showed whether there is problem with overall performance or with the deployment. 
<br>I could be due to lack of performance at the Underlay VMs themselves(not enough CPU, not enough Memory), lack of requests and limits defined at the pod level or requests and limits wrongly configured, it can be related to the Network on the node, can be from the application code itself.
<br>
**Best practices**
<br>
•	Review network diagram of all components for comparing the two measurable deployments (old and new)
<br>
•	Review conceptional diagram of all workloads, tech stack per workload and where they run on (APIs, web fronts etc) for the two measurable deployments (old and new)
<br>
•	Check test case scenario and the sample data used for performance test (how many concurrent users, how many test scenarios, iterations of test scenarios, sample data either random, or pre-recorded etc). 
<br>
•	If you observe that there are big variations in some operations (small, acceptable averages but big/slow perentiles90). Understand if these variations happen, randomly, or because of sample data but in a deterministic way, or because of caching etc. 
<br>
•	Dig into the full call stack of the calls noted as "External Call" in tools. 
<br>
•	Run the performance test for the environment (with the same sample data, and the same load) at least three times, to check whether there are significant variations per performance test cycle for the same environment (old, or new)
<br>
•	Gather Network stack traces and Process Dumps for given containers during the stress test (And not while manual / user testing). Find a way to maximize duration of capturing dumps.Capture [GCdump](https://devblogs.microsoft.com/dotnet/collecting-and-analyzing-memory-dumps/), capture [Processdump](https://npmsblog.wordpress.com/how-to-generate-iis-w3wp-process-dump-from-aks-windows-pod/), capture [tcpdump](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/packet-capture-pod-level) from the pod level.
<br>
•	Use the monitoring tabs in the Azure portal, also connect to the nodes/pods and collect [logs](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/identify-memory-saturation-aks)
• If in scope then guide customer to check code base as well.
<br>
<br>
<br>
Feedback appreciated :+1:               __[By Moumita Dey Verma](https://www.linkedin.com/in/moumita-dey-verma-8b61692a/)__  11/25/2022



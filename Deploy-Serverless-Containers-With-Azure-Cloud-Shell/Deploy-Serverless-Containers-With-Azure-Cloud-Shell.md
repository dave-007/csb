# How to use Build and Deploy Serverless ASP.Net Core Docker Application to Azure Container Instances With Azure's Cloud Shell

## a.k.a. Look Ma, No Dev Software, No Build Server, No App Server

## Better Title TBD

### Introduction

Ok, alright, enough about Azure, PowerShell, CLI, .Net Core, docker, containers, serverless... my head is exploding!
Can someone just walk me through all of these things in a single step by step tutorial?

#### NOTE FROM DAVE: This is an early draft for review by CloudSkills builders only, please don't share outside the group without discussing first. I think I have a lot to add to make this as clear as possible, including the following

### TODO

- A big picture overview diagram mapping out what we're doing here.
- Glossary of terms with links for more details.
- Consistency in naming things.
- Smooth way to handle 'expired token' or other causes of lost Azure Cloud Shell session.
- Decision whether to break into parts to make it more digestible
- Create settings using `code mysettings.ps1`, paste and edit, so no code editor needed.
- Move $tags up in tutorial
- YOUR TESTING FOR TECHNICAL CORRECTNESS AND CLARITY
- YOUR FEEDBACK, a pull request or Slack message.

### Multi-Part Outline for this topic

- Part 1 - Getting started, provision ACR
  - Azure Cloud Shell login
  - Bearings in Cloud Shell Azure
  - Use Code Editor to build settings file
    - Create settings using `code mysettings.ps1`, paste and edit, so no code editor needed.
  - Deploy ACR via ARM Template
- Part 2 - Build .Net Core Application, Dockerize and Push to ACR
  - dotnet new..
  - OR
  - git clone
  - Use Code Editor to build DOCKERFILE
    - Get into details of docker commands
      - 
  - az acr build
- Part 3 - Deploy ACR/Repo/Image/Tag container to ACI
  - Locate full path to image
  - Build params hash
  - Deploy to ACI
  - Review debug, logs, web app
  - Cleanup

### Terminology / Glossary

TBD

#### Goals

- Use several **Azure Cloud Shell** cloud native tools to build and deploy an application as a serverless container.
- Deploy infrastructure as code using a quickstart **ARM Template**.
- Create an **ASP.Net Core** application with no development environment setup.
- Use the **Azure Cloud Shell Editor** to create a docker file.
- Use **Azure CLI** *az acr build* command to build and push a docker image to your **Azure Container Registry** with no tooling setup.
- Deploy a Docker image as a serverless container to **Azure Container Instances** with no infrastructure setup.
- Use **Azure Cloud Shell** with **Azure PowerShell** and **Azure CLI** command line interface as powerful tools in infrastructure and application development and deployment.

#### In this guide you will learn to

- Perform all steps using [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview)
- Use [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/get-started-azureps?view=azps-3.4.0) to deploy an [ARM Quickstart Template](https://azure.microsoft.com/en-us/resources/templates/) to create an [Azure Container Registry (ACR)](https://azure.microsoft.com/en-us/services/container-registry/)
- Use [.Net] Core CLI](https://docs.microsoft.com/en-us/dotnet/core/tools/?tabs=netcore2x) to create a new asp.net core web application
- Use [Azure Cloud Shell Editor](https://docs.microsoft.com/en-us/azure/cloud-shell/using-cloud-shell-editor) to create a Docker file
- Use [Azure CLI](https://github.com/Azure/azure-cli) to build a Docker image and push it as an image to a repository in your ACR
- Use [Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/get-started-azureps?view=azps-3.4.0) to deploy your image as an [Azure Container Instance](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-overview) to run in a Serverless fashion.

## Prerequisites

in order to follow this guide, you will need:

- An active Azure subscription
- A text editor like **Visual Studio Code**, or **Notepad**

## Step 1 -- Get Into Azure Cloud Shell & Setup Variables

Log into your Azure account at [portal.azure.com](https://portal.azure.com), then click the shell icon in the top nav bar to enter the **Azure Cloud Shell**. Choose **PowerShell** as your shell type, and choose to create your cloud drive storage if this is your first time in the shell. First time users may refer to [David Lamb's Azure Cloud Shell tutorial (1.5 min)](https://www.youtube.com/watch?v=HBKm1-_kWKg) for guidance.

During the tutorial you may want maximize the cloud shell window using the maximize button at the upper right. Click that button again to restore the cloud shell window back down to the original size.

![Cloud Shell maximize/restore button](https://i.imgur.com/WRUhp4A.png)

Next you define variables for the shell commands to follow. Naming things in Azure can be tricky, with different naming rules for different types of resources. For simplicity, stick with lowercase letters and numbers.

<$>[note]
**Note:** Certain values must be GLOBALLY UNIQUE, like registry names and domain names, so something like 'DaveApp' doesn't usually work. It is a good idea to append numbers to a name, like _daveawesomedockerapp0123_ to create a unique name easily.
<$>

Copy the code below into your favorite text editor, modify the values, copy then and paste them into the cloud shell **using right click**, then press enter to run the last line if needed. This will define the variables you'll need for this tutorial.

```ps
# Your personal settings, customize these using lowercase letters and numbers
$myLocationName = 'eastus2'                 # An Azure region close to you, 'Get-AzLocation' to list them all
$myRGName       = 'containerdemo-rg'        # Resource group to contain this demo
$myWebAppName   = 'davewebappdocker0123'    # GLOBALLY UNIQUE name for your web app
$myRegistryName = 'daveregistry0123'        # GLOBALLY UNIQUE name for your container registry
$myDNSName      = 'daveawesomedockerapp0123'# GLOBALLY UNIQUE dns name, will be prepended to .azurecontainer.io

```

![Cloud shell right-click paste](https://i.imgur.com/ji418T6.gif)

<$>[warning]
**Warning:** The success of the tutorial depends on the variables above. If for some reason your Azure Cloud Shell session is interrupted, once you resume your session, run the code again to redefine the variables again to continue.
<$>

## Step 2 -- Deploy Azure Container Registry with an ARM Template

Next you will use `New-AzResourceGroup` to create a resource group for this demo.

```ps

New-AzResourceGroup -Name $myRGName -Location $myLocationName

```

Now you deploy your **Azure Container Registry** (ACR), once you have tested the name to ensure uniqueness.

```ps

Test-AzContainerRegistryNameAvailability -Name $myRegistryName

```

<$>[warning]
If _NameAvailable_ is not _True_ in the result, redefine your registry name by running the command `$myRegistryName = 'mynewregname001'` , then press **UP-Arrow** twice, scrolling through command history to display the `Test-AzContainerRegistryNameAvailability` command, then press **enter** to run it again.
<$>

With your registry name verified as unique, review and run the code below, which will set`$containerRegistryTemplateUrl` to the URL for one of the quickstart [Azure Quickstart Templates Templates](https://azure.microsoft.com/en-us/resources/templates/), also called **Azure Resource Manager Templates**, to define your **Azure Container Registry** resource. Then you define `$containerRegistryParams` as a [hash table](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_hash_tables?view=powershell-7) to pass the parameters the template needs. Then you execute `New-AZResourceGroupDeployment` to deploy the ARM template.

Paste the code block below to deploy.

```ps
    $containerRegistryTemplateUrl = 'https://raw.githubusercontent.com/microsoft/devops-project-samples/master/dotnet/aspnetcore/kubernetes/ArmTemplates/containerRegistry-template.json'
    $containerRegistryParams = @{
        registryName        = $myRegistryName
        registryLocation    = $myLocationName
    }
    New-AZResourceGroupDeployment -Name "$myRGName-ACR-Deployment" -ResourceGroupName $myRGName -TemplateUri $containerRegistryTemplateUrl -TemplateParameterObject $containerRegistryParams

```

If all is well, the `ProvisioningState` value should display `Succeeded`

![result of ARM template deployment](https://i.imgur.com/uQBQ0Gh.png)

<$>[Note]
**Note:** If at first you don't succeed, review the **Activity Log** under **Notifications**
![Troubleshooting ARM template deployments](https://i.imgur.com/7L7UTnJ.png)
<$>

Next you use the `Get-AZContainerRegistry` command to store your **Azure Container Registry** (ACR) information in a variable to refer to it later in the tutorial.

<$>[note]
**Note:** This command assumes a single registry exists, otherwise it uses the first one it finds.
<$>

```ps
$myACR = (Get-AZContainerRegistry)[0]
$myACR
```

![Result of `Get-AzContainerRegistry`](https://i.imgur.com/xLtJLm7.png)

## Step 3 -- Create a .Net Core Web Application

Before continuing, note the prompt for Azure Cloud Shell, `PS Azure:\>` . You can imagine this as the root directory of a file server, but instead of files it holds hierarchy of *all your Azure resources*. You can type `Get-ChildItem` to see your subscription(s), and use `Set-Location <TAB>` to begin navigating that hierarchy. But that is the subject of another tutorial.

Instead, type `cd ~\clouddrive` to navigate to your clouddrive , and type `pwd` (an alias of `Get-ChildItem`) to see your current location in the drive.

```ps
cd ~\clouddrive
pwd
```

Once there, create a new folder `docker-working` and `cd` into it.

```ps
mkdir docker-working
cd docker-working
```

![docker working directory](https://i.imgur.com/hxjpZwo.png)

Now you will build a new .Net Core web application using the `dotnet` command line tool.

<$>[note]
**Note:** Alternately, instead of creating a new application, you could use `git clone` to work with an existing application in github. This is another topic for a future tutorial.
<$>

Paste the next 3 lines to create and compile your new web app, then use `ls` to inspect the contents of the published folder.

```ps
dotnet new webapp -o mywebapp
cd mywebapp
dotnet publish -c Release  # TODO needed, now that docker is doing?

ls ./bin/Release/netcoreapp2.2/publish/
pwd
```

![Results of dotnet new](https://i.imgur.com/t7wCFHB.png)

## Step 4 - Create a Docker File

You can type `cls` to clear the screen, then type `code DOCKERFILE` to launce the cloud shell editor. Use the editor to paste in the content of the dockerfile below.

```ps
code DOCKERFILE
```

 Paste the following code block to the DOCKERFILE **using *Control-V* instead of right-click**, then click the ellipsis **(...)** in upper right corner of the cloud shell editor to save, then close editor.

```ps

FROM mcr.microsoft.com/dotnet/core/sdk:2.2 AS build-env
WORKDIR /app
# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore
# copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out
# build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:2.2-stretch-slim
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "mywebapp.dll"]

```

![Using Code Editor](https://i.imgur.com/qVYftbJ.gif)

## Step 5 - Build and Push a Docker Image to Azure Container Registry

First, run `pwd` to ensure you're in the root of the webapp folder, then use the [Azure CLI command `az acr build`](https://docs.microsoft.com/en-us/cli/azure/acr?view=azure-cli-latest#az-acr-build) command below (including the period at the end) to build the docker image. This is a powerful tool that enables you to build docker images without setting up or running docker on your workstation.

```ps

az acr build --registry $myACR.Name --image mywebapp:v1 .

```

<$>[warning]
**Warning:** If you get an error on this step, possibly your Azure Cloud Shell session was interrupted, you can repeat the code from step 2 above to define `$myACR` then try it again.
<$>

![Result of az acr build](https://i.imgur.com/8aQNgsD.gif)

## Step 6 - Prepare to Deploy Your Docker Image as a Serverless Container

Gather information needed for deploying to **Azure Container Instances** using the `New-AzContainerGroup` command.

First, get your container repository from your container registry (ACR):

```ps
[array]$myRepositories = az acr repository list --name $myACR.Name | ConvertFrom-Json  # Is there a AZ PowerShell way to get this?
$myRepository = $myRepositories[0]
$myRepository
```

Next, get the latest tag from the image in your container repository:

```ps
$myImageTags = az acr repository show-tags --name $myACR.Name --repository $myRepository --detail --orderby time_desc | ConvertFrom-JSON
$myImageTag = $myImageTags[0]
$myImageTag.name
```

Now build the path to the docker image you will deploy. Note the format is a DNS name that points to your **Azure Container Registry**, followed by the repository (**mywebapp**) and the tag (**v1**.)

```ps
$myDockerImagePath = "$($myACR.LoginServer)/$($myRepository):$($myImageTag.name)"
$myDockerImagePath
```

Since this is a private container registry, you need to provide credentials. Use the `Get-AzContainerRegistryCredential` command along with the `PSCredential` type accelerator to create the credential object.

```ps
$regCred = Get-AzContainerRegistryCredential -ResourceGroupName $myRGName  -Name $myACR.Name
$PSCred = [PSCredential]::New($regCred.Username, (ConvertTo-SecureString $regCred.Password -AsPlainText -Force)) # Type accelerators rule!
```

In this _optional_ step, you create a hash table to define resource tags, these are useful for keeping track of your Azure resources.

```ps
$tags = @{
    Env = 'Dev'
    Purpose = "Use Azure PowerShell command 'New-AzContainerGroup' to deploy containers from your Azure container registry easily!"
}
```

Now set up parameters in a hash table to pass to the command via [Splatting](https://devblogs.microsoft.com/scripting/use-splatting-to-simplify-your-powershell-scripts/)

```ps
$ContainerGroupArguments = @{
    ResourceGroupName   = $myRGName
    Name                = "$myWebAppName-cg"    #build container group name from the webapp name
    Image               = $myDockerImagePath
    IpAddressType       = 'Public'
    DNSNameLabel        = $myDNSName
    Port                = 80                    #optional
    RegistryCredential  = $PSCred               #required if this is NOT a public docker registry
    IdentityType        = 'SystemAssigned'      #optional, but useful for a future blog post
    Tag                 = $tags                 #optional uses hashtable defined above
    Debug               = $true                 #optional but very interesting to see the debug output
    }

# review contents of the parameter hash table
$ContainerGroupArguments
```

## Step 7 - Deploy Your container, Review and Test

You are ready to deploy your new Azure container group by running the `New-AzContainerGroup` command. Since it has a `Debug` flag in the arguments, you will need to confirm the execution. Review the debug output in yellow for an interesting peek under the hood of Azure.

```ps
$newACG = New-AzContainerGroup @ContainerGroupArguments
```

After you review the Debug output, inspect your container group via the `$newACG` variable.

```ps
$newACG | Get-Member
$newACG | Select-Object *

```

The `FQDN` property is the *Fully Qualified Domain Name* of your containerized application. Paste this into your favorite browser to see your new ASP.Net Core serveless containerized web application in action.

```ps
Write-Output " Paste    $($newACG.FQDN)  into your favorite web browser"
```

After browsing the web application, review the log of your container instance.

```ps
Get-AzContainerInstanceLog  -ResourceGroupName $myRGName -ContainerGroupName $myCGName
```

## Step 8 - Clean Up

The resources for this tutorial are very cost effective. Even so, it's a good idea to clean up once completed.

```ps
Remove-AzContainerGroup -Name $myCGName -ResourceGroupName $myRGName -Debug -Force
```

## Conclusion

You have used **Azure Cloud Shell** on a whirlwind tour of powerful tools and features to:

- Create an **ASP.Net Core** web application
- Build a **Docker** container
- Push that container to a repository image in your private **Azure Container Registry**
- Deploy that image to **Azure Container Instances**


Along the way you explored features of **Azure Cloud Shell** including **Azure PowerShell** and **Azure CLI**, **Azure Resource Manager**templates, the Cloud Shell Code Editor, and the .Net Core CLI.

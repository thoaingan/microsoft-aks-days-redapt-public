# Azure DevOps Pipelines

The `azure-pipelines.yml` file is setup to perform a docker build and then push to azure container registry.

Before this pipeline can be run, there are some pre-requisites.

## Pre-Requisites

### Fork the Favorite Beer repo 

In order to follow along with the Azure DevOps portions, and have some room to experiment, we recommend forking the favorite-beer repository to your github account.

On the github page for favorite-beer, on the rop right, select `Fork`. Choose your username if prompted, follow prompts.

### Azure Container Registry

Can be easily created via the Azure portal, or another scripting framework like Terraform or ARM.

In this case, we have set up a `redaptdemo` acr for the purposes of testing this build functionality.

#### Attach the Registry to our Cluster

Before running any az commands, you will need to have performed `az login`.

`az aks update -n jmeisnertest -g jm-rgp --attach-acr jmeisnertestacr`

### Azure Container Registry Service Connection

Go into Project settings on Azure DevOps and then navigate to `Service Connections`.

Click on `+ New service connection` and select `Docker Registry`, click `Next`.

Select `Azure Container Registry` from the options, and select your subscription in the dropdown.

In the registry drop down, select your container registry created earlier.

Call the connection `redaptdemo`. Make sure that `Grant access permission to all pipelines` is selected.

Click `Save`, and wait for the connection to be created.

You can also create custom service endpoints/connections.
https://docs.microsoft.com/en-us/azure/devops/extend/develop/service-endpoints?view=azure-devops

### Azure Kubernetes Service Environment + Service Connection

On the pipelines page, select `Create environment` name it `AKS Cluster Demo`, and select `Kubernetes` as the Resource, click `Next`.

Under Provider, select `Azure Kubernetes Service`.

Under Azure subscription, select your subscription, where the AKS cluster exists.

Under Cluster, slect your AKS cluster, that you would like to deploy to.

For Namespace, in this context, we will chose the `Existing` `default`.

Click the `Validate and create` button, to kick off the creation process.

This will automatically setup a Service Connection for our AKS cluster, which we will use later.

### Creating a Pipeline

In Azure DevOps, under your desired Project. Select `Pipelines`, and then choose `New Pipeline`.

Select `Github` from the present options. 

If you forked the repo earlier, you should see it at the top of the list, however you can search for `favorite-beer` if its not visible. Click into that repo, to begin the process.

If you are prompted to `Approve & Install Azure Pipelines` it should default to limited scope, choosing `Approve & Install` will allow Azure DevOps access to your repository.

Following the instructions on any prompts that occur for single-sign on services, you should land on a page with the title `Review your pipeline YAML`.

*Note: The default yaml that generates for your own applications would not include all of the steps here.*

```
# Docker
# Build and push an image to Azure Container Registry
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- master

resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '2a175992-247b-463c-8f5e-d20a95675f2c'
  imageRepository: 'favoritebeer'
  containerRegistry: 'jmeisnertestacr.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/spa-react-netcore-redis/voting/voting/Dockerfile'
  buildContextPath: '$(Build.SourcesDirectory)/spa-react-netcore-redis/voting'
  helmChartPath: '$(Build.SourcesDirectory)/spa-react-netcore-redis/voting/voting/k8s/Chart'
  valueFilePath: '$(Build.SourcesDirectory)/spa-react-netcore-redis/voting/voting/k8s/Chart/values.yaml'
  aksServiceConnection: 'd096e26f-577c-4fe9-9ec0-56c53100cec5'
  tag: '$(Build.BuildId)'
  latestTag: '$(Build.SourceBranchName)-latest'
  releaseName: 'demo'
  
  # Agent VM image name
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: Build and push stage
  jobs:  
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        buildContext: $(buildContextPath)
        tags: |
          $(tag)
          $(latestTag)

- stage: Deploy
  displayName: Deploy to Cluster
  jobs:  
  - job: Deploy
    displayName: Deploy
    pool:
      vmImage: $(vmImageName)
    steps:

    - task: HelmInstaller@1
      condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master'))
      displayName: Helm installer
      inputs: 
        helmVersionToInstall: 3.0.2

    - task: HelmDeploy@0
      condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/master'))
      displayName: Deploy to Demo
      inputs:
        connectionType: Kubernetes Service Connection
        kubernetesServiceEndpoint: $(aksServiceConnection)
        command: upgrade
        overrideValues: 'image.repository=$(containerRegistry)/$(imageRepository),image.tag=$(tag)'
        chartType: FilePath
        chartPath: $(helmChartPath)
        valueFile: $(valueFilePath)
        releaseName: $(releaseName)
        install: true
        failOnStderr: false
```

At this step, it should show you what is currently in the github, which may not match what you are seeing above.

What you should see is `tag: '$(Build.BuildId)'` under the variables section. Go ahead and add a prefix to this tag, with some initials, to identify the pipeline.

In my case, I change it to `tag: 'jm-$(Build.BuildId)'` for testing purposes, the main takeaway, is that you may want to choose a tag thats unique across pipelines, for example to test changes to the pipeline itself without impacting images being built on the mainline.

You will also see `dockerRegistryServiceConnection` and `containerRegistry` which will need to be changed to reflect the connection created previously.

The final step, would be to connect the AKS Service connection to our deploy step.

You can find the value for this on the `Project Settings -> Service Connections` page. 

If you created an environment, in the previous steps, You will see in this list a Service Connection that matches this format: `<EnvName>-<clusterName>-<nameSpace>-XXXXXXXXXXXX`, click into the details page.

In the browser, in the url bar, you will see `_settings/adminservices?resourceId=XXXX-XXXX-XXXX-XXXXX`. Grab the resourceId value and put it into your azure-pipelines.yaml for the following variable.

`aksServiceConnection: 'XXXX-XXXX-XXXX-XXXXX'`

If you made edits select `Save and run` twice, to confirm `Commit directly to the master branch`, if you did not, simply select `Run`.

*Note: If you did not fork this to your own repo, you wont be able to follow these steps, as you wont have write permission to the main repo.*

## Wrap up and run it.

All of the secrets and variables should work now, kick off a job! We are using simple logic, that can be run on the ubuntu-base image, but you can specify a custom image as well.

https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops#install

https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops#default-agent-pools

We have also included a step to push a packaged helm chart as an artifact, using an Azure DevOps built-in task. You can view others here: 

https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/?view=azure-devops
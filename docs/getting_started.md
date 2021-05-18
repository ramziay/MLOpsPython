# Getting Started with MLOpsPython <!-- omit in toc -->

This guide shows how to get MLOpsPython working with a sample ML project **_diabetes_regression_**. The project creates a linear regression model to predict diabetes and has CI/CD DevOps practices enabled for model training and serving when these steps are completed in this getting started guide.

If you would like to bring your own model code to use this template structure, follow the [custom model](custom_model.md) guide. We recommend completing this getting started guide with the diabetes model through ACI deployment first to ensure everything is working in your environment before converting the template to use your own model code.

- [Setting up Azure DevOps](#setting-up-azure-devops)
  - [Install the Azure Machine Learning extension](#install-the-azure-machine-learning-extension)
- [Get the code](#get-the-code)
- [Create a Variable Group for your Pipeline](#create-a-variable-group-for-your-pipeline)
  - [Variable Descriptions](#variable-descriptions)
- [Provisioning resources using Azure Pipelines](#provisioning-resources-using-azure-pipelines)
  - [Create an Azure DevOps Service Connection for the Azure Resource Manager](#create-an-azure-devops-service-connection-for-the-azure-resource-manager)
  - [Create the IaC Pipeline](#create-the-iac-pipeline)
- [Create an Azure DevOps Service Connection for the Azure ML Workspace](#create-an-azure-devops-service-connection-for-the-azure-ml-workspace)
- [Set up Build, Release Trigger, and Release Multi-Stage Pipeline](#set-up-build-release-trigger-and-release-multi-stage-pipelines)
  - [Set up the Model CI Training, Evaluation, and Registration Pipeline](#set-up-the-model-ci-training-evaluation-and-registration-pipeline)
  - [Set up the Release Deployment and/or Batch Scoring Pipelines](#set-up-the-release-deployment-andor-batch-scoring-pipelines)
- [Further Exploration](#further-exploration)
  - [Deploy the model to Azure Kubernetes Service](#deploy-the-model-to-azure-kubernetes-service)
    - [Web Service Authentication on Azure Kubernetes Service](#web-service-authentication-on-azure-kubernetes-service)
  - [Deploy the model to Azure App Service (Azure Web App for containers)](#deploy-the-model-to-azure-app-service-azure-web-app-for-containers)
  - [Example pipelines using R](#example-pipelines-using-r)
  - [Observability and Monitoring](#observability-and-monitoring)
  - [Clean up the example resources](#clean-up-the-example-resources)
- [Next Steps: Integrating your project](#next-steps-integrating-your-project)
  - [Additional Variables and Configuration](#additional-variables-and-configuration)
    - [More variable options](#more-variable-options)
    - [Local configuration](#local-configuration)

## Setting up Azure DevOps

You'll use Azure DevOps for running the multi-stage pipeline with build, model training, and scoring service release stages. If you don't already have an Azure DevOps organization, create one by following the instructions at [Quickstart: Create an organization or project collection](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops).

If you already have an Azure DevOps organization, create a new project using the guide at [Create a project in Azure DevOps and TFS](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=azure-devops).

### Install the Azure Machine Learning extension

Install the **Azure Machine Learning** extension to your Azure DevOps organization from the [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-air-aiagility.vss-services-azureml).

This extension contains the Azure ML pipeline tasks and adds the ability to create Azure ML Workspace service connections.

## Get the code

We recommend using the [repository template](https://github.com/microsoft/MLOpsPython/generate), which effectively forks the repository to your own GitHub location and squashes the history. You can use the resulting repository for this guide and for your own experimentation.

## Create a Variable Group for your Pipeline

MLOpsPython requires some variables to be set before you can run any pipelines. You'll need to create a _variable group_ in Azure DevOps to store values that are reused across multiple pipelines or pipeline stages. Either store the values directly in [Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=designer#create-a-variable-group) or connect to an Azure Key Vault in your subscription. Check out the [Add & use variable groups](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml#use-a-variable-group) documentation to learn more about how to create a variable group and link it to your pipeline.

Navigate to **Library** in the **Pipelines** section as indicated below:

![Library Variable Groups](./images/library_variable_groups.png)

Create a variable group named **`devopsforai-aml-vg`**. The YAML pipeline definitions in this repository refer to this variable group by name.

The variable group should contain the following required variables. **Azure resources that don't exist yet will be created in the [Provisioning resources using Azure Pipelines](#provisioning-resources-using-azure-pipelines) step below.**

| Variable Name            | Suggested Value           | Short description                                                                                                           |
| ------------------------ | ------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| BASE_NAME                | [your project name]       | Unique naming prefix for created resources - max 10 chars, letters and numbers only                                         |
| LOCATION                 | centralus                 | [Azure location](https://azure.microsoft.com/en-us/global-infrastructure/locations/), no spaces                             |
| RESOURCE_GROUP           | mlops-RG                  | Azure Resource Group name                                                                                                   |
| WORKSPACE_NAME           | mlops-AML-WS              | Azure ML Workspace name                                                                                                     |
| AZURE_RM_SVC_CONNECTION  | azure-resource-connection | [Azure Resource Manager Service Connection](#create-an-azure-devops-service-connection-for-the-azure-resource-manager) name |
| WORKSPACE_SVC_CONNECTION | aml-workspace-connection  | [Azure ML Workspace Service Connection](#create-an-azure-devops-azure-ml-workspace-service-connection) name                 |
| ACI_DEPLOYMENT_NAME      | mlops-aci                 | [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/) name                           |                 |


Make sure you select the **Allow access to all pipelines** checkbox in the variable group configuration.

More variables are available for further tweaking, but the above variables are all you need to get started with this example. For more information, see the [Additional Variables and Configuration](#additional-variables-and-configuration) section.

### Variable Descriptions

**BASE_NAME** is used as a prefix for naming Azure resources and should be unique. When sharing an Azure subscription, the prefix allows you to avoid naming collisions for resources that require unique names, for example, Azure Blob Storage and Registry DNS. Make sure to set BASE_NAME to a unique name so that created resources will have unique names, for example, MyUniqueMLamlcr, MyUniqueML-AML-KV, and so on. The length of the BASE_NAME value shouldn't exceed 10 characters and must contain letters and numbers only.

**LOCATION** is the name of the [Azure location](https://azure.microsoft.com/en-us/global-infrastructure/locations/) for your resources. There should be no spaces in the name. For example, central, westus, westus2.

**RESOURCE_GROUP** is used as the name for the resource group that will hold the Azure resources for the solution. If providing an existing Azure ML Workspace, set this value to the corresponding resource group name.

**WORKSPACE_NAME** is used for creating the Azure Machine Learning Workspace. You can provide an existing Azure ML Workspace here if you've got one.

**AZURE_RM_SVC_CONNECTION** is used by the [Azure Pipeline](../environment_setup/iac-create-environment-pipeline.yml) in Azure DevOps that creates the Azure ML workspace and associated resources through Azure Resource Manager. You'll create the connection in a [step below](#create-an-azure-devops-service-connection-for-the-azure-resource-manager).

**WORKSPACE_SVC_CONNECTION** is used to reference a [service connection for the Azure ML workspace](#create-an-azure-devops-azure-ml-workspace-service-connection). You'll create the connection after [provisioning the workspace](#provisioning-resources-using-azure-pipelines) in the [Create an Azure DevOps Service Connection for the Azure ML Workspace](#create-an-azure-devops-service-connection-for-the-azure-ml-workspace) section below.

**ACI_DEPLOYMENT_NAME** is used for naming the scoring service during deployment to [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/).


## Provisioning resources using Azure Pipelines

The easiest way to create all required Azure resources (Resource Group, Azure ML Workspace, Container Registry, and others) is to use the **Infrastructure as Code (IaC)** [pipeline with ARM templates](../environment_setup/iac-create-environment-pipeline-arm.yml) or the [pipeline with Terraform templates](../environment_setup/iac-create-environment-pipeline-tf.yml). The pipeline takes care of setting up all required resources based on these [Azure Resource Manager templates](../environment_setup/arm-templates/cloud-environment.json), or based on these [Terraform templates](../environment_setup/tf-templates).

**Note:** Since Azure Blob storage account required for batch scoring is optional, the resource provisioning pipelines mentioned above do not create this resource automatically, and manual creation is required before use.

### Create an Azure DevOps Service Connection for the Azure Resource Manager

The [IaC provisioning pipeline](../environment_setup/iac-create-environment-pipeline.yml) requires an **Azure Resource Manager** [service connection](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml#create-a-service-connection).

![Create service connection](./images/create-rm-service-connection.png)

Leave the **`Resource Group`** field empty.

**Note:** Creating the Azure Resource Manager service connection scope requires 'Owner' or 'User Access Administrator' permissions on the subscription.
You'll also need sufficient permissions to register an application with your Azure AD tenant, or you can get the ID and secret of a service principal from your Azure AD Administrator. That principal must have 'Contributor' permissions on the subscription.

### Create the IaC Pipeline

In your Azure DevOps project, create a build pipeline from your forked repository:

![Build connect step](./images/build-connect.png)

Select the **Existing Azure Pipelines YAML file** option and set the path to [/environment_setup/iac-create-environment-pipeline-arm.yml](../environment_setup/iac-create-environment-pipeline-arm.yml) or to [/environment_setup/iac-create-environment-pipeline-tf.yml](../environment_setup/iac-create-environment-pipeline-tf.yml), depending on if you want to deploy your infrastructure using ARM templates or Terraform:

![Configure step](./images/select-iac-pipeline.png)

If you decide to use Terraform, make sure the ['Terraform Build & Release Tasks' from Charles Zipp](https://marketplace.visualstudio.com/items?itemName=charleszipp.azure-pipelines-tasks-terraform) is installed.

Having done that, run the pipeline:

![IaC run](./images/run-iac-pipeline.png)

Check that the newly created resources appear in the [Azure Portal](https://portal.azure.com):

![Created resources](./images/created-resources.png)

## Create an Azure DevOps Service Connection for the Azure ML Workspace

At this point, you should have an Azure ML Workspace created. Similar to the Azure Resource Manager service connection, you need to create an additional one for the Azure ML Workspace.

Create a new service connection to your Azure ML Workspace using the [Machine Learning Extension](https://marketplace.visualstudio.com/items?itemName=ms-air-aiagility.vss-services-azureml) instructions to enable executing the Azure ML training pipeline. The connection name needs to match `WORKSPACE_SVC_CONNECTION` that you set in the variable group above (eg. 'aml-workspace-connection').

![Created resources](./images/ml-ws-svc-connection.png)

**Note:** Similar to the Azure Resource Manager service connection you created earlier, creating a service connection with Azure Machine Learning workspace scope requires 'Owner' or 'User Access Administrator' permissions on the Workspace.
You'll need sufficient permissions to register an application with your Azure AD tenant, or you can get the ID and secret of a service principal from your Azure AD Administrator. That principal must have Contributor permissions on the Azure ML Workspace.

## Set up Build, Release Trigger, and Release Multi-Stage Pipelines

Now that you've provisioned all the required Azure resources and service connections, you can set up the pipelines for training (CI) and deploying (CD) your machine learning model to production. Additionally, you can set up a pipeline for batch scoring.

1. **Model CI, training, evaluation, and registration** - triggered on code changes to master branch on GitHub. Runs linting, unit tests, code coverage, and publishes and runs the training pipeline. If a new model is registered after evaluation, it creates a build artifact containing the JSON metadata of the model. Definition: [diabetes_regression-ci.yml](../.pipelines/diabetes_regression-ci.yml).
1. **Release deployment** - consumes the artifact of the previous pipeline and deploys a model to either [Azure Container Instances (ACI)](https://azure.microsoft.com/en-us/services/container-instances/), [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service), or [Azure App Service](https://docs.microsoft.com/en-us/azure/machine-learning/service/how-to-deploy-app-service) environments. See [Further Exploration](#further-exploration) for other deployment types. Definition: [diabetes_regression-cd.yml](../.pipelines/diabetes_regression-cd.yml).
   1. **Note:** Edit the pipeline definition to remove unused stages. For example, if you're deploying to Azure Container Instances and Azure Kubernetes Service only, delete the unused `Deploy_Webapp` stage.
1. **Batch Scoring Code Continuous Integration** - consumes the artifact of the model training pipeline. Runs linting, unit tests, code coverage, publishes a batch scoring pipeline, and invokes the published batch scoring pipeline to score a model.

These pipelines use a Docker container on the Azure Pipelines agents to accomplish the pipeline steps. The container image ***mcr.microsoft.com/mlops/python:latest*** is built with [this Dockerfile](../environment_setup/Dockerfile) and has all the necessary dependencies installed for MLOpsPython and ***diabetes_regression***. This image is an example of a custom Docker image with a pre-baked environment. The environment is guaranteed to be the same on any building agent, VM, or local machine. In your project, you'll want to build your own Docker image that only contains the dependencies and tools required for your use case. Your image will probably be smaller and faster, and it will be maintained by your team.

### Set up the Model CI, training, evaluation, and registration pipeline

In your Azure DevOps project, create and run a new build pipeline based on the [diabetes_regression-ci.yml](../.pipelines/diabetes_regression-ci.yml)
pipeline definition in your forked repository.

If you plan to use the release deployment pipeline (in the next section), you will need to rename this pipeline to `Model-Train-Register-CI`.

Once the pipeline is finished, check the execution result:

![Build](./images/model-train-register.png)

And the pipeline artifacts:

![Build](./images/model-train-register-artifacts.png)

Also check the published training pipeline in the **mlops-AML-WS** workspace in [Azure Machine Learning Studio](https://ml.azure.com/):

![Training pipeline](./images/training-pipeline.png)

Great, you now have the build pipeline for training set up which automatically triggers every time there's a change in the master branch!

After the pipeline is finished, you'll see a new model in the **ML Workspace**:

![Trained model](./images/trained-model.png)

To disable the automatic trigger of the training pipeline, change the `auto-trigger-training` variable as listed in the `.pipelines\diabetes_regression-ci.yml` pipeline to `false`.  You can also override the variable at runtime execution of the pipeline.

The pipeline stages are summarized below:

#### Model CI

- Linting (code quality analysis)
- Unit tests and code coverage analysis
- Build and publish _ML Training Pipeline_ in an _ML Workspace_

#### Train model

- Determine the ID of the _ML Training Pipeline_ published in the previous stage.
- Trigger the _ML Training Pipeline_ and waits for it to complete.
  - This is an **agentless** job. The CI pipeline can wait for ML pipeline completion for hours or even days without using agent resources.
- Determine if a new model was registered by the _ML Training Pipeline_.
  - If the model evaluation determines that the new model doesn't perform any better than the previous one, the new model won't register and the _ML Training Pipeline_ will be **canceled**. In this case, you'll see a message in the 'Train Model' job under the 'Determine if evaluation succeeded and new model is registered' step saying '**Model was not registered for this run.**'
  - See [evaluate_model.py](../diabetes_regression/evaluate/evaluate_model.py#L118) for the evaluation logic.
  - [Additional Variables and Configuration](#additional-variables-and-configuration) for configuring this and other behavior.

#### Create pipeline artifact

- Get the info about the registered model
- Create a pipeline artifact called `model` that contains a `model.json` file containing the model information.

### Set up the Release Deployment and/or Batch Scoring pipelines

---
**PREREQUISITE**

In order to use these pipelines:

1. Follow the steps to set up the Model CI, training, evaluation, and registration pipeline.
1. You **must** rename your model CI/train/eval/register pipeline to `Model-Train-Register-CI`.

These pipelines rely on the model CI pipeline and reference it by name.

If you would like to change the name of your model CI pipeline, you must edit this section of yml for the CD and batch scoring pipeline, where it says `source: Model-Train-Register-CI` to use your own name.
```
trigger: none
resources:
  containers:
  - container: mlops
    image: mcr.microsoft.com/mlops/python:latest
  pipelines:
  - pipeline: model-train-ci
    source: Model-Train-Register-CI # Name of the triggering pipeline
    trigger:
      branches:
        include:
        - master
```

---

The release deployment and batch scoring pipelines have the following behaviors:

- The pipeline will **automatically trigger** on completion of the Model-Train-Register-CI pipeline for the master branch.
- The pipeline will default to using the latest successful build of the Model-Train-Register-CI pipeline. It will deploy the model produced by that build.
- You can specify a `Model-Train-Register-CI` build ID when running the pipeline manually. You can find this in the url of the build, and the model registered from that build will also be tagged with the build ID. This is useful to skip model training and registration, and deploy/score a model successfully registered by a `Model-Train-Register-CI` build.

### Set up the Release Deployment pipeline

In your Azure DevOps project, create and run a new build pipeline based on the  [diabetes_regression-cd.yml](../.pipelines/diabetes_regression-cd.yml)
pipeline definition in your forked repository.

Your first run will use the latest model created by the `Model-Train-Register-CI` pipeline.

Once the pipeline is finished, check the execution result:

![Build](./images/model-deploy-result.png)

To specify a particular build's model, set the `Model Train CI Build Id` parameter to the build Id you would like to use.

![Build](./images/model-deploy-configure.png)

Once your pipeline run begins, you can see the model name and version downloaded from the `Model-Train-Register-CI` pipeline.

![Build](./images/model-deploy-get-artifact-logs.png)

The pipeline has the following stage:

#### Deploy to ACI

- Deploy the model to the QA environment in [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/).
- Smoke test
  - The test sends a sample query to the scoring web service and verifies that it returns the expected response. Have a look at the [smoke test code](../ml_service/util/smoke_test_scoring_service.py) for an example.

### Set up the Batch Scoring pipeline

In your Azure DevOps project, create and run a new build pipeline based on the  [diabetes_regression-batchscoring-ci.yml](../.pipelines/diabetes_regression-batchscoring-ci.yml)
pipeline definition in your forked repository. 

Once the pipeline is finished, check the execution result:

![Build](./images/batchscoring-ci-result.png)

Also check the published batch scoring pipeline in the **mlops-AML-WS** workspace in [Azure Portal](https://portal.azure.com/):

![Batch scoring pipeline](./images/batchscoring-pipeline.png)

Great, you now have the build pipeline set up for batch scoring which automatically triggers every time there's a change in the master branch!

The pipeline stages are summarized below:

#### Batch Scoring CI

- Linting (code quality analysis)
- Unit tests and code coverage analysis
- Build and publish *ML Batch Scoring Pipeline* in an *ML Workspace*

#### Batch Score model

- Determine the model to be used based on the model name (required), model version, model tag name and model tag value bound pipeline parameters.
  - If run via Azure DevOps pipeline, the batch scoring pipeline will take the model name and version from the `Model-Train-Register-CI` build used as input.
  - If run locally without the model version, the batch scoring pipeline will use the model's latest version.
- Trigger the *ML Batch Scoring Pipeline* and waits for it to complete.
  - This is an **agentless** job. The CI pipeline can wait for ML pipeline completion for hours or even days without using agent resources.
- Use the scoring input data supplied via the SCORING_DATASTORE_INPUT_* configuration variables, or uses the default datastore and sample data.
- Once scoring is completed, the scores are made available in the same blob storage at the locations specified via the SCORING_DATASTORE_OUTPUT_* configuration variables.

To configure your own custom scoring data, see [Configure Custom Batch Scoring](custom_model.md#Configure-Custom-Batch-Scoring).


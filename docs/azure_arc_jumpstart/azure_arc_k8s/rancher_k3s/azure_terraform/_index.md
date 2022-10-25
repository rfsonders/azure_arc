---
type: docs
title: "k3s Azure Terraform plan"
linkTitle: "k3s Azure Terraform plan"
weight: 2
description: >
---

## Deploy Rancher k3s on an Azure VM and connect it to Azure Arc using Terraform

The following Jumpstart scenario will guide you on how to use the provided [Terraform](https://www.terraform.io/) plan to deploy a "Ready to Go" Azure virtual machine installed with single-master Rancher K3s Kubernetes cluster and connected it as an Azure Arc cluster resource.

## Prerequisites

- Clone the Azure Arc Jumpstart repository

    ```shell
    git clone https://github.com/microsoft/azure_arc.git
    ```

- [Install or update Azure CLI to version 2.36.0 and above](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest). Use the below command to check your current installed version.

  ```shell
  az --version
  ```

- [Install Terraform >=1.1.9](https://learn.hashicorp.com/terraform/getting-started/install.html)

- Create Azure service principal (SP)

    To be able to complete the scenario and its related automation, Azure service principal assigned with the “Contributor” role is required. To create it, login to your Azure account run the below command (this can also be done in [Azure Cloud Shell](https://shell.azure.com/)).

    ```shell
    az login
    subscriptionId=$(az account show --query id --output tsv)
    az ad sp create-for-rbac -n "<Unique SP Name>" --role "Contributor" --scopes /subscriptions/$subscriptionId
    ```

    For example:

    ```shell
    az login
    subscriptionId=$(az account show --query id --output tsv)
    az ad sp create-for-rbac -n "JumpstartArcK8s" --role "Contributor" --scopes /subscriptions/$subscriptionId
    ```

    Output should look like this:

    ```json
    {
    "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "displayName": "JumpstartArcK8s",
    "password": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    ```

    > **NOTE: If you create multiple subsequent role assignments on the same service principal, your client secret (password) will be destroyed and recreated each time. Therefore, make sure you grab the correct password**.

    > **NOTE: The Jumpstart scenarios are designed with as much ease of use in-mind and adhering to security-related best practices whenever possible. It is optional but highly recommended to scope the service principal to a specific [Azure subscription and resource group](https://docs.microsoft.com/cli/azure/ad/sp?view=azure-cli-latest) as well considering using a [less privileged service principal account](https://docs.microsoft.com/azure/role-based-access-control/best-practices)**

- [Enable subscription with](https://docs.microsoft.com/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider) the two resource providers for Azure Arc-enabled Kubernetes. Registration is an asynchronous process, and registration may take approximately 10 minutes.

  ```shell
  az provider register --namespace Microsoft.Kubernetes
  az provider register --namespace Microsoft.KubernetesConfiguration
  az provider register --namespace Microsoft.ExtendedLocation
  ```

  You can monitor the registration process with the following commands:

  ```shell
  az provider show -n Microsoft.Kubernetes -o table
  az provider show -n Microsoft.KubernetesConfiguration -o table
  az provider show -n Microsoft.ExtendedLocation -o table
  ```

## Automation Flow

For you to get familiar with the automation and deployment flow, below is an explanation.

1. User edits the tfvars and vars.sh script to match the environment.
2. User runs ```terraform init``` to download the required terraform providers.
3. User access the bootstrap VM created by the terraform plan and connects the K3s cluster to Azure Arc using the SPN credentials.
4. User verifies the Arc-enabled Kubernetes cluster.
5. User deploys a sample application.

## Deployment

The only thing you need to do before executing the Terraform plan is to create the tfvars file which will be used by the plan. This is based on the Azure service principal you've just created and your subscription.

- Navigate to the [terraform folder](https://github.com/microsoft/azure_arc/tree/main/azure_arc_k8s_jumpstart/rancher_k3s/terraform) and fill in the terraform.tfvars file with the values for your environment.

- Retrieve your Azure subscription ID using the ```az account list``` command.

- The Terraform plan execute a script on the VM OS to install all the needed artifacts as well to inject environment variables. Edit the [*scripts/vars.sh*](https://github.com/microsoft/azure_arc/blob/main/azure_arc_k8s_jumpstart/rancher_k3s/azure/terraform/scripts/vars.sh) to match the Azure service principal you created as well as the location and VM name that matches your environment.

- Run the ```terraform init``` command which will download the Terraform AzureRM provider.

    ![terraform init](./01.png)

- Run the ```terraform apply --auto-approve``` command and wait for the plan to finish.

    ![terraform apply completed](./02.png)

## Connecting to Azure Arc

> **NOTE: The VM bootstrap includes the log in process to Azure as well deploying the needed Azure Arc CLI extensions - no action items on you there!**

- SSH to the VM using the created Azure Public IP and your username/password.

    ![Azure VM public IP](./03.png)

- Check the cluster is up and running using the ```kubectl get nodes -o wide```

    ![k3s cluster nodes](./04.png)

- Using the Azure service principal you've created, run the below command to connect the cluster to Azure Arc.

    ```shell
    az connectedk8s connect --name <Name of your cluster as it will be shown in Azure> --resource-group <Azure resource group name> --correlation-id "d009f5dd-dba8-4ac7-bac9-b54ef3a6671a"
    ```

    For example:

    ```shell
    az connectedk8s connect --name arck3sdemo --resource-group Arc-K3s-Demo --correlation-id "d009f5dd-dba8-4ac7-bac9-b54ef3a6671a"
    ```

    ![Successful azconnctedk8s command](./05.png)

    ![Azure Arc-enabled Kubernetes cluster in an Azure resource group](./06.png)

    ![Azure Arc-enabled Kubernetes cluster in an Azure resource group](./07.png)

## K3s External Access

Traefik is the (default) ingress controller for k3s and uses port 80. To test external access to k3s cluster, an "*hello-world*" deployment was [made available](https://github.com/microsoft/azure_arc/blob/main/azure_arc_k8s_jumpstart/rancher_k3s/azure/terraform/deployment/hello-kubernetes.yaml) for you and it is included in the *home* directory [(credit)](https://github.com/paulbouwer/hello-kubernetes).

- Since port 80 is taken by Traefik [(read more about here)](https://github.com/rancher/k3s/issues/436), the deployment LoadBalancer was changed to use port 32323 along side with the matching Azure Network Security Group (NSG).

    ![Azure Network Security Group (NSG) rule](./08.png)

    ![hello-kubernetes.yaml file](./09.png)

- To deploy it, use the ```kubectl apply -f hello-kubernetes.yaml``` command. Run ```kubectl get pods``` and ```kubectl get svc``` to check that the pods and the service has been created.

    ![kubectl apply -f hello-kubernetes.yaml command](./10.png)

    ![kubectl get pods command](./11.png)

    ![kubectl get svc command](./12.png)

- In your browser, enter the *cluster_public_ip:32323* which will bring up the *hello-world* application.

    ![hello-kubernetes application in a web browser](./13.png)

## Delete the deployment

- The most straightforward way is to delete the cluster is via the Azure Portal, just select cluster and delete it.

    ![Delete Azure Arc-enabled Kubernetes cluster](./14.png)

- If you want to nuke the entire environment, just delete the Azure resource group or alternatively, you can use the ```terraform destroy --auto-approve``` command.

    ![Delete Azure resource group](./15.png)

    ![terraform destroy](./16.png)

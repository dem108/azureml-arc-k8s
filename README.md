# Azure ML - Arc - K8s

Test run Azure Arc enabled Machine Learning.

## How to

Run this on Windows 10. For Kubernetes, use Docker Desktop and enable Kubernetes.

### Prerequisites

Because we're using local machine (with GPU) for this scenario, we need following:

- Local Windows 10 machine with GPU
- WSL 2 (needed for enhanced performance for Docker)
- Docker Desktop for Windows
- Visual Studio Code
- Windows Terminal (optional)
- Powershell (to install Chocolatey, used to install Helm on Windows)
- An Azure subscription (check out [creating a free account](https://azure.microsoft.com/free) if you don't have it already)
- Azure ML workspace

### Steps

1. Use VSCode and open terminal, or use Windows terminal when needed

1. Set up CLI

    1. [Install Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) and/or [Update Azure CLI](https://docs.microsoft.com/en-us/cli/azure/update-azure-cli)) by `az upgrade`. Check current version by `az version`. Make sure to have the required version.

    1. Install extensions

        - `connectedk8s` extension (it should be >= 1.0.0):

          ```
          az extension add --name connectedk8s
          ```

        - `k8s-extension` extension:

          ```cmd
          az extension add --name k8s-extension
          ```

    1. Login (you will need it later but you can verify here too)

          ```cmd
          az login
          az account set --subscription <your_subscription_id>
          ```

    1. Set up defaults for CLI for convenience

       ```cmd
       az configure --defaults group=<your-resource-group> workspace=<your-AML-workspace>
       ```

       For example,

       ```cmd
       az configure --defaults group=rg202001-aml-handson workspace=aml-handson-202001
       ```

1. Set up local Kubernetes

    1. Enable Kubernetes in Docker Desktop for Windows.

        1. Check out [Docker Desktop documentation on Resource](https://docs.docker.com/docker-for-windows/#resources) to configure memory assignment etc.

    1. Install [latest release of Helm 3](https://helm.sh/docs/intro/install).

        1. Install [Chocolatey](https://chocolatey.org/) first to install Helm on Windows. Follow [install guide](https://chocolatey.org/install). After Chocolatey install, `choco install kubernetes-helm`. Both Chocolatey and Helm install will require you to run PowerShell as admin.

1. Set up an Azure Arc cluster

    1. You will need 'Read' and 'Write' permission on `Microsoft.Kubernetes/connectedClusters`.

    1. Azure Arch agents will need to [meet network requirements](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster#meet-network-requirements)

    1. Register providers

        ```cmd
        az provider register --namespace Microsoft.Kubernetes
        az provider register --namespace Microsoft.KubernetesConfiguration
        az provider register --namespace Microsoft.ExtendedLocation

        az provider show -n Microsoft.Kubernetes -o table
        az provider show -n Microsoft.KubernetesConfiguration -o table
        az provider show -n Microsoft.ExtendedLocation -o table
        ```

    1. Create a resource group

        ```cmd
        az group create --name AzureArcSEA --location southeastasia --output table
        ```

    1. Connect an existing Kubernetes cluster (which is the Kubernetes cluster running locally in your machine). This will take a few minutes.

        ```cmd
        az connectedk8s connect --name AzureArcSEA1 --resource-group AzureArcSEA
        ```

    1. Verify the cluster (and visit Azure Portal too)

        ```cmd
        az connectedk8s list --resource-group AzureArcSEA --output table
        ```

    1. View Azure Arc agents for Kubernetes

        ```cmd
        kubectl get deployments,pods -n azure-arc
        ```

    1. (later) You can clean up resources. Note: This removes both associated configuration resources and agents running on the cluster. If you use Azure Portal to remove it, agents on the cluster will remain. Best practice is to use `az connectedk8s delete` instead of Azure Portal.

        ```cmd
        az connectedk8s delete --name AzureArcSEA1 --resource-group AzureArcSEA
        ```

    1. ~~Get `kubeconfig` from somewhere (TBU).~~

1. Check versions by `az version` and confirm they meet [Azure Arc enabled Kubernetes cluster extensions prerequisites](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/extensions#prerequisites).

    1. Azure CLI version >= 2.24.0

    1. Azure CLI k8s-extension extension version >= 0.4.3

1. Deploy Azure ML extension

    1. `az login` and `az account set --subscription <sub_id>` if not done already.

    1. Kick off AML extension deployment

        ```cmd
        az k8s-extension create --name amlarc-compute --extension-type Microsoft.AzureML.Kubernetes --configuration-settings enableTraining=True --cluster-type connectedClusters --cluster-name <your-connected-cluster-name> --resource-group <resource-group> --scope cluster
        ```

       For example,

        ```cmd
        az k8s-extension create --name amlarc-compute --extension-type Microsoft.AzureML.Kubernetes --configuration-settings enableTraining=True --cluster-type connectedClusters --cluster-name AzureArcSEA1 --resource-group AzureArcSEA --scope cluster
        ```

    1. Verify deployment (ensure all are running) - this takes longer to finish.

        ```cmd
        kubectl get pods -n azureml
        ```

1. Attach Arc cluster in Azure ML workspace

    1. Use [studio](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-attach-arc-kubernetes#attach-arc-cluster-studio) or [Python SDK](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-attach-arc-kubernetes#attach-arc-cluster-python-sdk).

        1. Optionally, you can use JSON configuration file for advanced attachment scenario: setting memory footage etc. General guidence is [here](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-attach-arc-kubernetes#advanced-attach-scenario).

1. Finally, run samples

    1. [Image classification model with AML 2.0 CLI](https://github.com/Azure/AML-Kubernetes/blob/master/docs/simple-train-cli.md)

## Resources

- [Azure ML - How to attach Arc Kubernetes](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-attach-arc-kubernetes)

- [AML-Kubernetes GitHub samples](https://github.com/Azure/AML-Kubernetes)

- To use Kubernetes dashboard, try [this](kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml), start the server with `kubectl proxy`, and visit [here](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)
    - to bypass authentication for testing purposes, check this [blog](https://andrewlock.net/running-kubernetes-and-the-dashboard-with-docker-desktop/).

- [Azure Arc K8s troubleshooting guide](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/troubleshooting?WT.mc_id=Portal-Microsoft_Azure_Health)

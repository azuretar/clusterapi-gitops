## Manage your Kubernetes Clusters with Cluster API, Azure Arc and GitOps 
In this session we are going to Introduce Cluster API, a Kubernetes subproject that allows you to manage Kubernetes clusters lifecycle running anywhere using only Kubernetes YAML files. Let’s see how Azure Arc GitOps approach improves and simplify the day-2 operations of these clusters, where your Git repo is now the source of truth. Do you have problems managing identities and Network connection for your current CI/CD process? You don’t know how to manage multiple Kubernetes clusters in production? Then this talk/repo is for you!

Slide Deck: https://www.slideshare.net/JorgeArteiro/manage-your-kubernetes-cluster-with-cluster-api-azure-and-git-ops

Meetup reference: https://www.meetup.com/en-AU/Microsoft-Reactor-Sydney/events/279879195

Follow us at https://youtube.com/AzureTar , https://AzureTar.com and  @AzureTar
### Scripts are grouped the following way:

(Dependencies) - All environment/installation scripts required.

(Management Cluster) - Cluster API management/control plane cluster creation, configuration and operations.

(Workload cluster) - Workload Clusters creation, configuration and operations using CAPIZ(Azure Provider https://capz.sigs.k8s.io/). 

(General) - Assorted scripts and commands.

(Reference Links) - Useful links to go deeper on Kubernetes Cluster API

### (Dependencies) Install Azure CLI (az) 
    curl -L https://aka.ms/InstallAzureCli | bash

### (Dependencies) Install Clusterctl
    curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v0.4.4/clusterctl-linux-amd64 -o clusterctl
    clusterctl version
    chmod +x ./clusterctl
    sudo mv ./clusterctl /usr/local/bin/clusterctl

### (Dependencies) Install Kubernetes CLIs
    az aks install-cli

### (Dependencies) Install Helm3 CLI
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh

### (Dependencies) Install/Update Extensions
    az extension list -o table

    az upgrade  (to upgrade all installed extensions)

    az extension add -n connectedk8s  or  az extension update -n connectedk8s

    az extension add -n k8s-configuration  or  az extension update -n k8s-configuration

    az extension add -n aks-preview  or  az extension update -n aks-preview
### (Management Cluster) Create AKS - Azure Kubernetes Services to install Cluster API management
    Create Azure resource Group on eastus regions where GitOps preview is available
    az group create -l eastus -n capi-controlplane
    
    Create Azure Kubernetes Services (Edit Script with your IDs)
    az aks create --resource-group capi-controlplane --name capi-controlplane \
        --node-count 1 --node-vm-size Standard_DS2_v2 \
        --network-plugin azure --network-policy calico \
        --enable-addons monitoring,azure-policy \
        --enable-managed-identity --generate-ssh-keys \
        --vm-set-type VirtualMachineScaleSets --zones 1 2 3 --load-balancer-sku standard \
        --enable-aad --aad-admin-group-object-ids "<AdminGroupObjectId>" \
        --max-pods 110 \
        --yes 

### (Management Cluster) Get AKS Management Cluster .kubeconfig Credential. Config will be merged on the ~/.kube/config file
    az aks get-credentials --resource-group capi-controlplane --name capi-controlplane

    kubectl get nodes (to test connection)

### (Management Cluster) Connect AKS control plane to Azure Arc
    az feature register --namespace Microsoft.ContainerService --name AKS-GitOps

    az provider register --namespace Microsoft.ContainerService

    az provider register --namespace Microsoft.KubernetesConfiguration

    az feature show --namespace Microsoft.ContainerService --name AKS-GitOps (make sure it's Registered)

    az aks enable-addons -a gitops -n capi-controlplane -g capi-controlplane

    az connectedk8s connect --name capi-controlplane --resource-group capi-controlplane --location eastus  
### (Management Cluster) Add GitOps Configuration to deploy workload cluster from YAML files, --git-path=clusters
    az k8s-configuration create \
        --name capi-controlplane --cluster-name capi-controlplane --resource-group capi-controlplane \
        --operator-instance-name capi-controlplane --operator-namespace default \
        --repository-url https://github.com/azuretar/clusterapi-gitops \
        --scope cluster --cluster-type managedClusters \
        --operator-params "--git-poll-interval 3s --git-readonly --git-path=clusters/ --git-branch main"

### (Workload cluster) Edit and Run arc_capi_azure.sh bash script to Initialize CAPI control plane and create workload cluster.
    git clone https://github.com/azuretar/clusterapi-gitops.git
    cd clusterapi-gitops/init

    (parameters: azuretar-reactor-1 is the cluster name, and true is to Initialize the CAPI control plane)
    . ./arc_capi_azure.sh azuretar-reactor-1 true
    mv azuretar-reactor-1.yaml ../clusters/ (Workload cluster will be maintained by Azure Arc GitOps)

    ps: to create extras clusters, call script with false at the end. 
    . ./arc_capi_azure.sh azuretar-reactor-2 false
    mv azuretar-reactor-2.yaml ../clusters/  (Workload cluster will be maintained by Azure Arc GitOps)

    ps: If script fails, stop and run again. 

    Based on JumpStart https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/cluster_api/capi_azure/

### (Workload cluster) Use --kubeconfig created by Init Script to connect the workload cluster
    kubectl --kubeconfig=./azuretar-reactor-1.kubeconfig get pods -A

    ps: do not push .kuconfig files to git repo. Please include *.kubeconfig in your .gitignore file

### (Workload cluster) Add GitOps Configuration to deploy workload application from YAML files, --git-path=workloads
    az k8s-configuration create \
        --name azuretar-reactor-1 --cluster-name azuretar-reactor-1 --resource-group azuretar-reactor-1 \
        --operator-instance-name azuretar-reactor-1 --operator-namespace default \
        --repository-url https://github.com/azuretar/clusterapi-gitops \
        --scope cluster --cluster-type connectedClusters \
        --operator-params "--git-poll-interval 3s --git-readonly --git-path=workloads/ --git-branch main"

    kubectl --kubeconfig=./azuretar-reactor-1.kubeconfig get pods -n default -w
### (Workload cluster) Install Azure Arc Extension to Azure Monitoring from az cli
    az k8s-extension create --name azuremonitor-containers --cluster-name azuretar-reactor-1 \
    --resource-group azuretar-reactor-1 \
    --cluster-type connectedClusters --extension-type Microsoft.AzureMonitor.Containers  

### (General) Using clusterctl commands
    clusterctl describe cluster azuretar-reactor-1

    kubectl get cluster --all-namespaces

    kubectl get kubeadmcontrolplane --all-namespaces

    kubectl delete cluster azuretar-reactor-1 (to clean up resources)

### (Reference Links)
https://github.com/azuretar/clusterapi-gitops

https://youtu.be/jYe1Dj1oGcc (Microsoft Reactor Talk recordings for the repo)

https://www.youtube.com/playlist?list=PLM4Db0UWu45LgXEwbW3PVgQ3iT77H8Bwg

https://cluster-api.sigs.k8s.io/user/concepts.html

https://cluster-api.sigs.k8s.io/user/quick-start.html

https://capz.sigs.k8s.io/

https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/cluster_api/capi_azure/

https://www.weave.works/technologies/gitops/

https://doc.crds.dev/github.com/kubernetes-sigs/cluster-api

https://doc.crds.dev/github.com/kubernetes-sigs/cluster-api-provider-azure@v0.4.13

https://github.com/kubernetes-sigs/image-builder

https://github.com/Azure/azure-capi-cli-extension

https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-use-gitops-connected-cluster

https://docs.microsoft.com/en-gb/azure/azure-arc/kubernetes/azure-rbac

https://docs.microsoft.com/en-gb/azure/azure-arc/kubernetes/cluster-connect

https://docs.microsoft.com/en-gb/azure/azure-monitor/containers/container-insights-enable-arc-enabled-clusters

https://docs.microsoft.com/en-gb/azure/aks/policy-reference

https://docs.microsoft.com/en-gb/azure/azure-arc/kubernetes/policy-reference

https://github.com/Azure/arc-k8s-demo

https://www.youtube.com/watch?v=hnLeAFnAJaM&t=1086s

https://github.com/azuretar/clusterapi-templates

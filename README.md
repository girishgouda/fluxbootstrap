export GITHUB_TOKEN=token

export RESOURCE_GROUP_NAME=gitops-demo-rg
export LOCATION=westeurope
export CLUSTER_NAME=GitOpsDemoCluster

az group create -n $RESOURCE_GROUP_NAME -l $LOCATION

az aks create -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME --enable-managed-identity

az aks get-credentials -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME

kubectl get nodes

flux check --pre

flux bootstrap github \
--owner girishgouda \
--personal true \
--repository fluxbootstrap

flux check

kubectl -n flux-system get gitrepositories.source.toolkit.fluxcd.io

kubectl -n flux-system get kustomizations.kustomize.toolkit.fluxcd.io

git clone git@github.com:girishgouda/fluxbootstrap.git

export RESOURCE_GROUP_NAME=gitops-demo-rg
export LOCATION=westeurope
export CLUSTER_NAME=GitOpsDemoCluster

RESOURCE_GROUP_ID=$(az group show -n $RESOURCE_GROUP_NAME -o tsv --query id)
AKS_RESOURCE_GROUP_NAME=$(az aks show -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME -o tsv --query nodeResourceGroup)
AKS_RESOURCE_GROUP_ID=$(az group show -n "MC_gitops-demo-rg_GitOpsDemoCluster_westeurope" -o tsv --query id)
KUBELET_CLIENT_ID=$(az aks show -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME -o tsv --query identityProfile.kubeletidentity.clientId)


az role assignment create --role "Virtual Machine Contributor" --assignee 03aecd0d-59f1-40b4-8c18-bc3a2c1bfd6f --scope /subscriptions/526be93c-8b93-4ca3-a34f-559d10cdcef4/resourceGroups/MC_gitops-demo-rg_GitOpsDemoCluster_westeurope

az role assignment create --role "Managed Identity Operator" --assignee  03aecd0d-59f1-40b4-8c18-bc3a2c1bfd6f --scope /subscriptions/526be93c-8b93-4ca3-a34f-559d10cdcef4/resourceGroups/gitops-demo-rg

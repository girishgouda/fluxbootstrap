export GITHUB_TOKEN=xxxxxx

export RESOURCE_GROUP_NAME=gitops-demo
export LOCATION=southeastasia
export CLUSTER_NAME=gitops
export GITHUB_USER=girishgouda
export GITHUB_REPO=fluxbootstrap

az group create -n $RESOURCE_GROUP_NAME -l $LOCATION

az aks create -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME --enable-managed-identity

az aks get-credentials -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME

kubectl get nodes

kubectl create ns flux-system


flux bootstrap github \
--owner=$GITHUB_USER \
--repository=$GITHUB_REPO \
--branch=main \
--path=./clusters/$CLUSTER_NAME

flux create kustomization demoapp \
  --namespace=flux-system \
  --source=flux-system \
  --path="./manifests" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./clusters/$CLUSTER_NAME/demoapp-kustomization.yaml

  mkdir manifests

cat > ./manifests/namespace.yaml <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: demoapp
EOF

cat > ./manifests/deployment.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp
  namespace: demoapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: "mcr.microsoft.com/dotnet/core/samples:aspnetapp"
          ports:
            - containerPort: 80
              protocol: TCP
EOF

cat > ./manifests/service.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: demoapp
  namespace: demoapp
spec:
  type: ClusterIP
  selector:
    app: demoapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF


RESOURCE_GROUP_ID=$(az group show -n $RESOURCE_GROUP_NAME -o tsv --query id)
AKS_RESOURCE_GROUP_NAME=$(az aks show -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME -o tsv --query nodeResourceGroup)
AKS_RESOURCE_GROUP_ID=$(az group show -n gitops-demo -o tsv --query id)
KUBELET_CLIENT_ID=$(az aks show -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME -o tsv --query identityProfile.kubeletidentity.clientId)

az role assignment create --role "Virtual Machine Contributor" --assignee 1c0d298d-03a0-4909-aadc-57ecf3b4f062 --scope /subscriptions/526be93c-8b93-4ca3-a34f-559d10cdcef4/resourceGroups/gitops-demo

az role assignment create --role "Managed Identity Operator" --assignee 1c0d298d-03a0-4909-aadc-57ecf3b4f062 --scope /subscriptions/526be93c-8b93-4ca3-a34f-559d10cdcef4/resourceGroups/gitops-demo

cat > ./manifests/aad-pod-identity.yaml <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: aad-pod-identity
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: aad-pod-identity
  namespace: aad-pod-identity
spec:
  url: https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
  interval: 10m
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: aad-pod-identity
  namespace: aad-pod-identity
spec:
  interval: 5m
  chart:
    spec:
      chart: aad-pod-identity
      version: 4.0.0
      sourceRef:
        kind: HelmRepository
        name: aad-pod-identity
        namespace: aad-pod-identity
      interval: 1m
  values:
    nmi:
      allowNetworkPluginKubenet: true
EOF


az identity create -n SopsDecryptorIdentity -g $RESOURCE_GROUP_NAME -l $LOCATION

CLIENT_ID=$(az identity show -n SopsDecryptorIdentity -g $RESOURCE_GROUP_NAME -o tsv --query "clientId")
OBJECT_ID=$(az identity show -n SopsDecryptorIdentity -g $RESOURCE_GROUP_NAME -o tsv --query "principalId")
RESOURCE_ID=$(az identity show -n SopsDecryptorIdentity -g $RESOURCE_GROUP_NAME -o tsv --query "id")

export KEY_VAULT_NAME=vaultgitops3

az keyvault create --name $KEY_VAULT_NAME --resource-group "gitops-demo" --location "southeastasia"

az keyvault key create --name sops-key --vault-name $KEY_VAULT_NAME --protection software --ops encrypt decrypt

az keyvault set-policy --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP_NAME --object-id $OBJECT_ID --key-permissions encrypt decrypt???

az keyvault key show --name sops-key --vault-name $KEY_VAULT_NAME --query key.kid

"https://vaultgitops3.vault.azure.net/keys/sops-key/3809539ab2504ecca7c1c6dfb671ff94"

cat > ./clusters/gitops/sops-identity.yaml <<EOF
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: sops-akv-decryptor
  namespace: flux-system
spec:
  clientID: 605bf726-78e7-4ab0-b3fe-182a6b8d4db8
  resourceID: /subscriptions/526be93c-8b93-4ca3-a34f-559d10cdcef4/resourcegroups/gitops-demo/providers/Microsoft.ManagedIdentity/userAssignedIdentities/SopsDecryptorIdentity
  type: 0 # user-managed identity
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: sops-akv-decryptor-binding
  namespace: flux-system
spec:
  azureIdentity: sops-akv-decryptor
  selector: sops-akv-decryptor  # kustomize-controller label will match this name
EOF???


SIGNED_IN_USER_OBJECT_ID=$(az ad signed-in-user show -o tsv --query objectId)
az keyvault set-policy --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP_NAME --object-id $SIGNED_IN_USER_OBJECT_ID --key-permissions encrypt decrypt
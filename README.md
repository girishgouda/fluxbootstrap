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
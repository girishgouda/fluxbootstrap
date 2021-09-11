export GITHUB_TOKEN=xxxxxx

export RESOURCE_GROUP_NAME=gitops-demo-rg
export LOCATION=westeurope
export CLUSTER_NAME=gitopsdemocluster
export GITHUB_USER=girishgouda
export GITHUB_REPO=fluxbootstrap


flux bootstrap github \
--owner=$GITHUB_USER \
--repository=$GITHUB_REPO \
--branch=main \
--path=./clusters/$CLUSTER_NAME